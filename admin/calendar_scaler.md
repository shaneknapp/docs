# Calendar Node Pool Autoscaler

A new GKE node takes about ten minutes to join the cluster and pull a user image.
A class logging in together waits all of it. The node placeholder scaler avoids
that by parking a low-priority pod on a spare node ahead of time. Real users evict
the placeholder, and the node is already warm.

Each placeholder nearly fills a node, so one placeholder is one idle node held in
reserve. How many to hold comes from a baseline in the support chart and from
events on a shared Google Calendar.

## Where it lives

| Piece | Where |
|---|---|
| Source | [berkeley-dsep-infra/jupyterhub-k8s-node-placeholder](https://github.com/berkeley-dsep-infra/jupyterhub-k8s-node-placeholder) |
| Image | `us-central1-docker.pkg.dev/cal-icor-hubs/core/node-placeholder-scaler` |
| Chart | `node-placeholder-scaler`, pulled from `oci://us-central1-docker.pkg.dev/cal-icor-hubs/helm-charts` and pinned in [`support/requirements.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/support/requirements.yaml) |
| Config | the `node-placeholder-scaler:` block in [`support/values.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/support/values.yaml) |
| Running as | `support-node-placeholder-scaler`, `support` namespace |
| Placeholders | `<pool>-placeholder` deployments, also in `support` |

The scaler is a subchart of the support chart, so it deploys on the support layer
with cert-manager, ingress-nginx, Prometheus and Grafana. See
[the deployment pipeline](deploy_pipeline).

## How it works

The pod runs `python3 -m scaler` in a loop, one pass every 60 seconds. Each pass
reads the config and placeholder template from a mounted ConfigMap, fetches the
calendar, works out a replica count per pool, and applies a `<pool>-placeholder`
deployment with `kubectl`.

The config is re-read every pass, so a `helm upgrade` that rewrites the ConfigMap
lands within a minute or two. The pod never needs restarting.

### Deciding the replica count

For each pool under `nodePools`, in order:

1. If the calendar asks for more than zero and `calendarOverrideEnabled` is true,
   that number wins outright.
2. Otherwise, if one of the pool's placeholder pods is `Pending`, the count holds
   at the configured `replicas`. A real user just evicted that placeholder and it
   has not landed yet, so scaling down here would throw the spare away.
3. Otherwise the count is the target minus one per node with room to spare,
   floored at zero. The target is the calendar count if there is one, the
   configured `replicas` if not.

A node has room when its free resource ratio is above the threshold for the
configured strategy:

| `scalingStrategy` | A node counts as free when |
|---|---|
| `cpu` | free CPU ratio > `cpuThreshold` |
| `mem` | free memory ratio > `memoryThreshold` |
| `balanced` | both are above their thresholds |

Four things keep a node from counting, even when it looks free:

- it already hosts a placeholder pod
- it is cordoned/unschedulable
- the scaler has been watching it for less than `nodeGracePeriod` seconds
- it was last busy less than `nodeGracePeriod` seconds ago

Both grace periods exist because a node's usage swings hard right after it appears
and right after a user logs out. Without them the scaler tears down a placeholder
it is about to need, and the count oscillates.

## The calendar

[Cal-ICOR Scaling Events](https://calendar.google.com/calendar/ical/c_35d90f50598c1472a4154c538bb49a21eabd8be93831d7de345d53fea8e19390%40group.calendar.google.com/public/basic.ics)
is a public Google calendar shared with infrastructure staff. The event
description is a YAML mapping of pool name to replica count:

``` yaml
base: 3
workshop: 1
```

The keys are pool names from `nodePools` in `support/values.yaml`, not hub names.
Today that means `base` and `workshop`. Unknown keys and non-integer counts are
dropped silently.

Scope the events tightly. If the surge is a 9am class, run the event from 8:50 to
9:10. When two events overlap and both name the same pool, the larger count wins.

An event asking for `0` skips the override path, which only fires above zero, but
it still sets the target to zero, so the count lands there anyway. That is how the
standing evening cool-off event works. Only a pending placeholder pod beats it,
holding at the configured `replicas`.

## Configuration

The config lives in the support helm chart, under `node-placeholder-scaler:` in
[`support/values.yaml`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/support/values.yaml).

| Key | Set to | What it does |
|---|---|---|
| `calendarUrl` | the scaling events feed | public ical feed. Omit it and the scaler says so in the log and runs on config counts alone |
| `calendarTimezone` | `America/Los_Angeles` | sets `TZ` in the container. Match the calendar's own timezone, or all-day events land in the wrong window |
| `calendarOverrideEnabled` | `True` | lets a calendar entry pin the count instead of only raising the target |
| `scalingStrategy` | `mem` | one of `cpu`, `mem`, `balanced` |
| `memoryThreshold` | `0.40` | free memory ratio above which a node counts as having room |
| `cpuThreshold` | commented out | unused under the `mem` strategy. Our workloads are memory bound |
| `nodeGracePeriod` | chart default, 600 | seconds a node is protected from reduction, when new and when recently freed |
| `nodePools.<name>.nodeSelector` | a `hub.jupyter.org/pool-name` value | which GKE pool the placeholder targets |
| `nodePools.<name>.resources.requests.memory` | bytes | the placeholder pod's size |
| `nodePools.<name>.replicas` | `base: 1`, `workshop: 0` | baseline count when the calendar is quiet |

A higher `memoryThreshold` means fewer nodes count as free, so placeholders
survive longer and a replacement node starts sooner. `0.40` is deliberately
aggressive: our users are memory bound and arrive in bursts.

### Sizing a placeholder pod

The placeholder has to be big enough that a user pod cannot fit beside it, and
small enough to still schedule alongside a node's system pods. Run the script in
the scaler repo against a node from the pool you are sizing:

``` bash
./tools/determine_placeholder_pod_memory.py <node name>
```

``` text
Node allocatable memory: 61483356160 bytes
Total non-notebook memory used by pods: 466964736 bytes

Recommended placeholder pod size for values.yaml: 60738518784 bytes
```

Put the recommended number into that pool's `resources.requests.memory`.

## Changing the configuration

Edit `support/values.yaml` on a feature branch and open a PR against `staging` in
`cal-icor-hubs`. The path picks up the `support-deployment` label, and merging
redeploys the support chart.

In term the `base` baseline is normally `1`. Over breaks, or for a pool nobody is
using, set it to `0`.

## Changing the scaler itself

The scaler lives upstream, in
[berkeley-dsep-infra/jupyterhub-k8s-node-placeholder](https://github.com/berkeley-dsep-infra/jupyterhub-k8s-node-placeholder).
Fork it, branch, and open a PR against `main`. It wants Python 3.11:

``` bash
conda create -n scalertest python=3.11
conda activate scalertest
pip install -r dev-requirements.txt
pip install -r node-placeholder-scaler/requirements.txt
pre-commit install
```

The tests cover `scaler.py` and `calendar_parser.py`, and run on every PR through
[`test-scaler.yaml`](https://github.com/berkeley-dsep-infra/jupyterhub-k8s-node-placeholder/blob/main/.github/workflows/test-scaler.yaml).
To run them yourself:

``` bash
python node-placeholder-scaler/tests/run_tests.py
```

To add or update a Python dependency, edit `requirements.in` and recompile from
inside `node-placeholder-scaler/`:

``` bash
pip-compile --output-file=requirements.txt requirements.in
```

Merging to `main` runs
[`build-publish-node-placeholder.yaml`](https://github.com/berkeley-dsep-infra/jupyterhub-k8s-node-placeholder/blob/main/.github/workflows/build-publish-node-placeholder.yaml),
which builds and pushes the image with `chartpress`, then pushes the packaged
chart to GAR. Chartpress derives the version from git history, hence
`0.0.1-0.dev.git.424.h1c64a4b`.

:::{admonition} That workflow is currently broken
:class: warning
It authenticates as a service account that no longer exists, so it fails. Until
the scaler's CI is repointed at keyless WIF, build and push by hand from a
workstation with GAR access:

``` bash
chartpress --push
helm package helm/node-placeholder
helm push node-placeholder-scaler-<version>.tgz oci://us-central1-docker.pkg.dev/cal-icor-hubs/helm-charts
```

Update the version for our deployment by editing the `node-placeholder-scaler`
entry in `support/requirements.yaml`. CI runs `helm dep up support` before
deploying.

## Monitoring

Tail the scaler:

``` bash
kubectl -n support logs -l app.kubernetes.io/name=node-placeholder-scaler -f
```

Every pass logs its reasoning, pool by pool:

``` text
2026-07-28 15:34:39,615 Found 0 events at https://calendar.google.com/calendar/...
2026-07-28 15:34:39,615 Overrides: {}
2026-07-28 15:34:39,615 Processing the node pool: base ...
2026-07-28 15:34:39,615 Node gke-spring-2025-user-pool-...-988j has 0.96 CPU free ratio and 0.99 Memory free ratio.
2026-07-28 15:34:40,093 Node gke-spring-2025-user-pool-...-988j has been observed for 0s, within 600s grace period. Skipping reduction.
2026-07-28 15:34:40,577 Calendar replica count for pool base: 0
2026-07-28 15:34:40,577 Config replica count for pool base: 1
2026-07-28 15:34:40,578 Pending placeholder pod detected for pool base: False
2026-07-28 15:34:40,578 Reducing base placeholder deployment replicas by 0 based on node resources.
2026-07-28 15:34:40,578 Final replica count for pool base: 1
2026-07-28 15:34:40,772 deployment.apps/base-placeholder configured
```

Check what it did:

``` bash
kubectl -n support get deploy | grep placeholder
kubectl -n support get pods -l component=placeholder -o wide
```

The placeholder pods run `registry.k8s.io/pause` at a negative priority class, so
they are first out when a real user pod needs the space.
