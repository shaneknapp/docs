# The Hub/Cluster Deployment Pipeline

Everything on the `spring-2025` cluster deploys from one workflow:
[`deploy-spring-2025.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/workflows/deploy-spring-2025.yaml).

There are four layers that run in a fixed order:

``` bash
cluster (tofu) -> support -> nfs -> hub
```

Each layer is its own reusable workflow. A layer runs only if every layer before
it succeeded or was skipped. The whole stack shares one concurrency group, so a
second merge queues behind the first instead of racing it.

The normal path is to open a PR, let the labeler label it by path, and merge.
That deploys the layers those labels name.

## Branch model

- `staging` is the shared branch and owns the cluster's shared infra.
- `prod` accepts merges from `staging` only, enforced by
  [`prevent-prod-merges.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/workflows/prevent-prod-merges.yaml).
- All work goes feature branch to PR to `staging`.

Cluster, support and NFS are `shared_branch_only`, so they deploy from `staging`
only. There is one support chart and one NFS server per cluster, not one per
environment. Merging `staging` into `prod` deploys hubs to prod and nothing else.

## Labels

The [labeler](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/labeler.yml)
adds these automatically, based on the paths you changed.

| Label | Layer | Added when the PR touches |
|---|---|---|
| `tofu-spring-2025` | cluster | `tofu/clusters/spring-2025/**` |
| `support-deployment` | support | `support/**` |
| `jupyterhub-home-nfs-deployment` | nfs | `jupyterhub-home-nfs/**` |
| `jupyterhub-deployment` | hub, all | `hub/**` |
| `hub-images` | hub, all | `images/**` |
| `hub: <name>` | hub, one | `deployments/<name>/**` |

CI never deploys `template`, as it's used for creating new deployments.

:::{admonition} Labels are applied by path, not by intent
:class: warning
The labeler can't tell a comment-only edit from a real change. A typo fix under
`tofu/clusters/spring-2025/` still gets `tofu-spring-2025`, and merging it runs a
real `apply`.

Manually strip the deploy-driving labels before you merge a docs-only or
comment-only PR. The gate then reports `skip` for those layers.
:::

## The gate

The `gate` job decides which layers run and publishes the decision as outputs the
other jobs read. It runs
[`decide-layers.py`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/scripts/decide-layers.py)
against a layer spec written inline in the workflow. On a push the decision comes
from the merged PR's labels plus the branch. On a dispatch the inputs pass
straight through.

Two important things to note:

1. A layer can't deploy into a cluster that isn't there. The gate checks that the
cluster is `RUNNING` before deciding anything. If it isn't, and this run isn't
creating it, the support, NFS and hub layers are forced off and the run prints a
notice saying which ones and why. Your change stays committed and deploys the
next time the stack comes up.

2. The hub layer is always on. It has no label of its own at the gate; the hub
workflow resolves which hubs from the labels instead. When there are no hub
labels the list is empty and the deploy job skips.

## Layers

### cluster

[`deploy-spring-2025-cluster.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/workflows/deploy-spring-2025-cluster.yaml)
runs Terragrunt over `tofu/clusters/spring-2025`.

It always plans first, with `-detailed-exitcode`:

| Exit | Meaning | Result |
|---|---|---|
| `1` | plan failed | job fails |
| `0` | no changes | apply skipped, notice printed |
| `2` | changes | apply runs |

The control plane was created by hand and isn't tofu-managed, so this layer
touches only the network and the node pools.

### support

[`deploy-spring-2025-support.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/workflows/deploy-spring-2025-support.yaml)
installs the support chart into the `support` namespace: cert-manager,
ingress-nginx, Prometheus, Grafana, statsd, and the
[node placeholder scaler](calendar_scaler).

It decrypts `support/secrets.yaml` with SOPS, logs in to the GAR Helm registry,
runs `helm upgrade --install --wait`, then waits on every rollout.

### nfs

[`deploy-spring-2025-nfs.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/workflows/deploy-spring-2025-nfs.yaml)
installs the `jupyterhub-home-nfs` chart. See [user storage](user_storage) for
what it does and how to operate it.

This one skips `helm --wait` on purpose. The chart runs a single replica and
allows it to be unavailable, which makes Helm's readiness check pass with zero
ready pods, so `kubectl rollout status` is the real gate.

### hub

[`deploy-spring-2025-hub.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/workflows/deploy-spring-2025-hub.yaml)
deploys hubs with `hubploy`.

It resolves the hub list with
[`determine-hub-deployments.py`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/scripts/determine-hub-deployments.py).
`jupyterhub-deployment` or `hub-images` means every hub; otherwise each
`hub: <name>` label contributes one. The list becomes a matrix, one runner per
hub, with `fail-fast` disabled so a broken hub doesn't cancel the rest.

Per hub:

``` bash
hubploy --verbose deploy --timeout 30m <hub> hub <staging|prod>
```

It then waits on every rollout in the `<hub>-<environment>` namespace.

## Running it by hand

The top-level workflow has a `workflow_dispatch` trigger. Pick a cluster command
(`skip`, `plan` or `apply`), check off the chart layers you want, and name a hub
and environment for the hub layer. Use it for a plan-only look at the
infrastructure, or to redeploy one hub without opening an empty PR. Each leaf
workflow has its own dispatch trigger if you only want that layer.

## Tearing the cluster down

None of the dispatch menus offer a destroy. The only path is merging a PR
labelled `tofu-destroy-spring-2025` into `staging`, which keeps a teardown
something you have to write down, label and get reviewed.

A destroy forces every other layer off in the same run. It removes the node pools
and the network, which is an outage, and leaves the control plane alone.

## Authentication

There are no service account keys anywhere in the pipeline or the repo. CI uses
[Workload Identity Federation](https://docs.cloud.google.com/iam/docs/workload-identity-federation):
GitHub mints an OIDC token for the run, and GCP exchanges it for short-lived
credentials.

Two identities, kept separate on purpose:

| Identity | Deploys | Fenced by |
|---|---|---|
| `prod-infra@` | cluster | the `prod-infra` GitHub environment |
| `prod-deploy@` | support, nfs, hub | `cluster-admin` RBAC inside `spring-2025` only |

`prod-deploy@`'s project IAM is read-only. GCP can't scope the `container.*`
roles to a single cluster, so the fence has to live on the Kubernetes side.

SOPS still decrypts secrets at deploy time, and `prod-deploy@` holds decrypt on
the `jupyterhubs/sops` KMS key.

## When something doesn't deploy

Start at the `gate` job, which prints its decision as one line:

``` bash
cluster_command: skip  deploy_support: false  deploy_nfs: false  deploy_hub: true  hub_environment: staging
```

If a layer shows `false` or `skip`, check these in order of likelihood:

1. The label was missing from the PR, or you stripped it. Check the merged PR.
2. You merged to `prod`, where cluster, support and NFS skip entirely.
3. The cluster was down. Look for the "Layers not deployed" notice in the run.
4. An earlier layer failed. Every layer needs the ones before it to have
   succeeded or skipped.

## The dev cluster

[`deploy-dev.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/workflows/deploy-dev.yaml)
is a parallel copy of this stack, built from the same script and the same layer
model, with its own `*-dev` labels and its own SOPS key. It exists for testing
changes to the pipeline itself against a throwaway cluster.

## Wiring up a new stack workflow

A label deploys nothing on its own. It has to name a layer in some stack
workflow's spec, and each cluster gets its own stack. See
[cluster infrastructure](infrastructure) for the tofu units and the labels
themselves; this is the workflow half.

Copy `deploy-dev.yaml` rather than the prod one. It is the same layer model with
fewer prod-only exceptions, and it already covers cluster creation, which
`spring-2025` predates.

### The orchestrator

Copy `deploy-dev.yaml` to `deploy-<cluster>.yaml` and change:

- `concurrency.group` to `<cluster>-stack`, so this cluster serializes against
  itself and not against another one.
- The `DEV_*` variable references in the gate to your cluster's prefix.
- Every `label:` in the `LAYER_SPEC` to the labels you created.
- Every `uses: ./.github/workflows/deploy-dev-<layer>.yaml` to your own leaves.

Drop the `nfs-volume` layer unless you need it. It exists on dev because a
recreated NFS server can come back on a different ClusterIP, and the PV's
`spec.nfs` is immutable.

The layer spec is read by
[`decide-layers.py`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/.github/scripts/decide-layers.py),
whose docstring defines every field (`label`, `always_on`, `shared_branch_only`,
`when_on`/`when_off`, `implied_by`, `requires`, `destroy_label`). Pass `-d` to
have it print the decision as YAML in the run log, the way the prod stack does.

Give the cluster layer a `destroy_label` only if you want a merge-to-teardown
path. Dev has none; its teardown is a dispatch on the leaf.

### The leaves

One reusable workflow per layer, each with a `workflow_call` trigger and its own
`concurrency.group`. For the cluster leaf, point `tg_dir` at
`tofu/clusters/<cluster>` and set `environment:` to the GitHub environment that
fences the infra identity.

A `uses:` pointing at a workflow that isn't on the branch fails the entire run
before any `if:` is evaluated, so add all the leaves in the same PR as the
orchestrator.

### Variables and identities

Create a repo variable set for the new prefix, mirroring `DEV_*` and `PROD_*`:

| Variable | Used by |
|---|---|
| `<X>_CLUSTER`, `<X>_ZONE`, `<X>_PROJECT` | the gate's cluster probe |
| `<X>_WIF_PROVIDER` | every job that authenticates |
| `<X>_DEPLOY_SA` | the probe and the chart layers |
| `<X>_INFRA_SA` | the cluster layer |

The deploy identity needs `container.clusterViewer` for the probe, plus a
`cluster-admin` RBAC binding inside the new cluster for the chart layers. Create
the GitHub environment (`<cluster>-infra`) before the first run. The cluster leaf
names it, and a missing environment fails the job.

### Two things that bite

The probe flag has to match the cluster's shape. `spring-2025` is regional and
passes `--location`; dev is zonal and passes `--zone`. Both read a variable named
`*_ZONE`, so `PROD_ZONE` actually holds a region.

Nothing is wired until the labels exist and appear in the spec. Open the PR with
the tofu units, the labeler entries, the orchestrator and the leaves together,
then dispatch the stack with `cluster_command: plan` before letting a merge run
an apply.
