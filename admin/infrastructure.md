# Cluster Infrastructure

The cluster's networking and node pools are managed as code with
[OpenTofu](https://opentofu.org/), orchestrated by
[Terragrunt](https://terragrunt.gruntwork.io/), in the
[`tofu/`](https://github.com/cal-icor/cal-icor-hubs/tree/staging/tofu) directory
of `cal-icor-hubs`.

Working on it needs `opentofu` and `terragrunt` installed locally. The
[cal-icor-hubs README](https://github.com/cal-icor/cal-icor-hubs#installing-other-software-packages-required-to-deploy-infrastructure)
covers both.

Managed here: the VPC and subnet, Cloud Router, Cloud NAT, reserved egress IP,
IAP-SSH firewall rule, every node pool, and the remote-state bucket.

Hubs and Helm charts belong to the [deployment pipeline](deploy_pipeline).

## Layout

``` bash
tofu/
  root.hcl                  shared remote state + common inputs
  modules/                  reusable code, no state of its own
    cluster/                GKE cluster (default pool removed)
    network/                Cloud Router, NAT, egress IP, IAP-SSH firewall
    nodepools/              private GKE node pool
    tfstate-bucket/         the state bucket itself
  bootstrap/
    tfstate-bucket/         global, not tied to a cluster
  clusters/
    cluster-template/       copy this to stand up a new cluster
    spring-2025/            the live prod cluster
    dev/                    throwaway cluster for testing the pipeline
```

## Units and state

Each leaf `terragrunt.hcl` is a *unit*. It includes `root.hcl` and points at a
module:

``` hcl
include "root" {
  path           = find_in_parent_folders("root.hcl")
  merge_strategy = "deep"
}

terraform {
  source = "../../../modules/nodepools"
}
```

Each unit has one state file. The unit's path under `tofu/` becomes its GCS
state prefix, so `tofu/clusters/spring-2025/network` stores state at
`clusters/spring-2025/network`.

Pool units are named for their role (`user-pool/`) rather than the date,
so the directory stays put while the pool name inside it changes.

`root.hcl` supplies the project (`cal-icor-hubs`) and region (`us-central1`) to
every unit, and generates `versions.tf` and `provider.tf` for modules that don't
ship their own. Per-cluster identity lives in `cluster.hcl` at the top of each
cluster folder, read by every unit below it, so the cluster name is written once.

## The state bucket

`bootstrap/tfstate-bucket` manages `cal-icor-hubs-tofu-state`, the bucket that
holds every unit's state, including its own.

`prevent_destroy` and `force_destroy = false` are set in the module, so a stray
destroy can't delete the bucket everything else's state lives in.

## Private nodes and NAT

Every node pool is private, so nodes have internal IPs only and outbound traffic
leaves through Cloud NAT. The `network` unit reserves a static egress IP,
`35.254.232.174`, giving everything the nodes pull (quay.io, ghcr.io, Docker Hub,
PyPI, conda, apt, ACME) one stable, allowlistable source address.

Two details to know before touching that unit:

- NAT covers `ALL_SUBNETWORKS_ALL_IP_RANGES`, including the GKE pod secondary
  range. Pod egress is the whole reason NAT is there, so covering only node
  primary IPs would break it.
- Dynamic port allocation is on. A multi-tenant JupyterHub node runs many pods
  opening outbound connections, and without it you get source-port exhaustion,
  which surfaces as intermittent egress failures that are hard to diagnose.

The inbound ingress IP is a separate address, pinned to the ingress-nginx
LoadBalancer in `support/values.yaml`. The two point in opposite directions.

SSH is IAP only, through the `spring-2025-allow-iap-ssh` rule (range
`35.235.240.0/20`, tcp:22, target tag `hub-cluster`).

## Node pools

One Terragrunt unit per pool, all sourcing `modules/nodepools`. The module
defaults carry the shared config; each unit sets only what differs, usually
machine type, disk, min/max nodes and labels.

`user-pool` is the only tainted pool (`hub.jupyter.org_dedicated=user:NO_SCHEDULE`),
so only singleuser servers and placeholder pods land there. It scales to zero,
though the [placeholder scaler](calendar_scaler) usually keeps a node warm.

Current machine types, sizes and what runs where are in the pool table in
[`tofu/README.md`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/tofu/README.md),
kept next to the code.

## Running a unit locally

Terragrunt drives OpenTofu, so point `TG_TF_PATH` at `tofu` first:

``` bash
export TG_TF_PATH=tofu
cd tofu/clusters/spring-2025/<unit>
terragrunt init
terragrunt plan
terragrunt apply
```

`terragrunt` has to be on your `PATH`; the pre-commit hooks need it too. An
`apply` mutates real infrastructure, so read the plan first.

To drive the whole cluster at once, run from the cluster directory:

``` bash
cd tofu/clusters/spring-2025
terragrunt run --all plan
```

## Deploying through CI

There is no separate tofu CI workflow. Plan, apply and destroy all run through
the `cluster` layer of the deployment pipeline, which drives
`terragrunt run --all` over `clusters/spring-2025`. CI pins OpenTofu `1.12.3` and
Terragrunt `1.1.0`.

It authenticates as `prod-infra@` with keyless Workload Identity Federation, a
different identity from the one that deploys the charts. See
[the deployment pipeline](deploy_pipeline) for the plan/apply exit-code logic,
the label that triggers it, and the destroy path.

The tofu pre-commit hooks (`tofu_fmt`, `terragrunt_fmt`,
`terragrunt_hcl_validate`, `terraform-docs-go`) run locally only. They sit under
`ci.skip`, so pre-commit.ci does not run them on PRs.

## Standing up a new cluster

Copy the template and edit it:

``` bash
cp -r tofu/clusters/cluster-template tofu/clusters/<cluster-name>
cd tofu/clusters/<cluster-name>
```

Next, set the `cluster_name` in `clusters/<cluster-name>/terragrunt.hcl` and
uncomment the block under `Redeploy`.

Then set the `pool_name` and double-check the `machine_type` in each of the
pool units.

Terragrunt walks the `dependency` graph, so the cluster comes up first, then the
network, then the pools. That ordering is wired in because NAT has to exist
before the private nodes do.

The template covers two cases, chosen in `cluster/terragrunt.hcl`:

| | New dev cluster | Redeploying an existing one |
|---|---|---|
| `create_network` | `true`, its own VPC | `false`, existing VPC |
| IP ranges | fresh, non-colliding | reuse the live cluster's |
| `deletion_protection` | `false`, so CI can tear it down | `true` |

A dev cluster gets its own VPC so its Cloud NAT and firewall rules don't collide
with prod's. A redeploy reuses the live ranges because the `jupyterhub-home-nfs`
client allowlist is written against them.

`spring-2025` itself doesn't follow this pattern. It has no `cluster/` unit and
runs on the `default` VPC, because the cluster already existed when the tofu
conversion happened.

Confirm the configs are valid before opening a PR:

``` bash
export TG_TF_PATH=tofu
cd tofu/clusters/<cluster-name>
terragrunt run --all plan
```

### Labels

The pipeline is label-driven, so a cluster's labels are part of standing it up.
They go on the same branch as the cluster definition.

Add the entries to
[`.github/labeler.yml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/labeler.yml).
One new label for the cluster:

``` yaml
# live tofu units for the <cluster-name> cluster
tofu-<cluster-name>:
  - 'tofu/clusters/<cluster-name>/**'
```

Then extend the existing functional labels with the new unit paths instead of
minting per-cluster copies of each. A PR touching one pool then carries both the
cluster label and the functional one:

``` yaml
tofu-network:
  - 'tofu/modules/network/**'
  - 'tofu/clusters/spring-2025/network/**'
  - 'tofu/clusters/<cluster-name>/network/**'
```

Create the repo labels themselves with `gh`. The labeler auto-creates a label it
can't find, but with a random color, so pre-creating them is how you control the
color and description. `7B42BC` is the tofu family purple:

``` bash
gh label create "tofu-<cluster-name>" \
  --color 7B42BC \
  --description "Live tofu units for the <cluster-name> cluster"
```

A cluster that needs a teardown path gets a second label, in red, matching
`tofu-destroy-spring-2025`:

``` bash
gh label create "tofu-destroy-<cluster-name>" \
  --color B60205 \
  --description "Merge with this label to tear down the <cluster-name> tofu units"
```

A label on its own deploys nothing. It has to appear as the `label:` (and
`destroy_label:`) of a cluster layer in a stack workflow's layer spec, the way
`tofu-spring-2025` does in `deploy-spring-2025.yaml`. Each cluster gets its own
stack workflow; see "Wiring up a new stack workflow" in
[the deployment pipeline](deploy_pipeline).

### The PR

Open it against `staging`, with the units, the labeler entries, and any workflow
wiring all in the one PR.

The labeler applies `tofu-<cluster-name>` by path when you open it. Paste the
`terragrunt run --all plan` output into the description so the reviewer can see
what will be created without running it themselves.

On merge to `staging`, the cluster layer plans again in CI and applies. Watch
the run: a first-time cluster build takes a while, and the pools can't come up
until the cluster and the NAT do.
