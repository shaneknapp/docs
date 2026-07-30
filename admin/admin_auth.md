# Administrator Authentication

## Prerequisites

### IAM and GCP Roles

Someone with IAM admin on the cal-icor-hubs GCP project adds you with
`roles/owner` and `roles/compute.admin`.

``` bash
roles/compute.admin
roles/owner
```

### Software Packages

Working installs of:

- [gcloud](https://cloud.google.com/sdk/docs/install)
- [gh](https://cli.github.com/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [sops](https://github.com/getsops/sops/releases)

For `sops`, `gcloud`, `gh` and `kubectl`, follow the links and install appropriately.

## Authenticating with `gcloud`, `kubectl` and `gh`

First, install the Google Cloud SDK, then the GKE auth plugin. `kubectl` 1.26+
will not talk to GKE without it and the error message does not say so clearly.

``` bash
gcloud components install gke-gcloud-auth-plugin
```

### GCP

There are two separate logins to the GCP project, and both are required:

``` bash
gcloud auth login                          # credentials for the gcloud CLI itself
gcloud auth application-default login      # ADC, a different credential store
gcloud config set project cal-icor-hubs    # set the default project
```

`gcloud auth login` only authenticates `gcloud`. Application Default
Credentials are what sops uses to reach the KMS key, and what `hubploy` and
Terragrunt use.

### GitHub CLI

``` bash
gh auth login
gh auth status
```

From the cloned `cal-icor-hubs` repository, in the root folder, set the default
repo that `gh` will talk to:

``` bash
gh repo set-default
```

Select the `upstream` (`cal-icor/cal-icor-hubs`) as the default.

### `kubectl`

The prod cluster is regional, not zonal, so this needs --region:

``` bash
gcloud container clusters get-credentials spring-2025 --region us-central1
```

That writes the context gke_cal-icor-hubs_us-central1_spring-2025 and makes it current. There is one cluster:

``` bash
$ gcloud container clusters list --project cal-icor-hubs
NAME         LOCATION     MASTER_VERSION      MASTER_IP      MACHINE_TYPE  NODE_VERSION        NUM_NODES  STATUS   STACK_TYPE
spring-2025  us-central1  1.35.5-gke.1241004  35.226.117.80  n1-highmem-8  1.35.5-gke.1241004  4          RUNNING  IPV4
```

Some of fields may vary (nodes) depending on how many resources are deployed.

Next, verify that everything works:

``` bash
gcloud auth list                                    # your account, starred
gcloud config get-value project                     # cal-icor-hubs
kubectl config current-context                      # gke_cal-icor-hubs_us-central1_spring-2025
kubectl get nodes                                   # 4 nodes
kubectl get ns | grep prod                          # one <hub>-prod per hub, 27 hubs
kubectl get pods -n jupyter-prod                    # hub, proxy, user pods
helm list -n jupyter-prod                           # the release

```
