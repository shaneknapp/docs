# Removing an Existing Hub Deployment

Sadly, an institution will decide to end their partnership with us, or not use
their hub after signing up.

Instead of having these hubs sit around unused, we should audit our deployments
at the end of every term, and remove any idle hubs that we're certain won't be
used in the future.

This is currently a manual process, but will need to eventually be automated.

## Prerequisites

### Software Packages and Authentication

You will need the basic set of admin tooling and authentication set up as
described in the [Creating a New Hub](new_hub) document.

## The canonical list of steps to remove a hub deployment

1. [Delete the alerts](#delete-the-alert-policy).
2. [Delete the `prod` and `staging` Helm deployments](#delete-the-helm-deployments).
3. [Archive or delete the `prod` folder on the NFS server](#archive-or-delete-nfs-storage).
4. [Remove deployment from GitHub labeler action](#remove-deployment-from-labeler-action).
5. [Remove deployment folder under `cal-icor-hubs/deployments/`](#remove-deployment-from-hub-repo).
6. [Delete GitHub labels and URLs in the Issue templates](#update-github).
7. [Review local changes and create a PR](#review-your-changes).
8. [Review and merge](#review-and-merge) the changes from steps (5) and (6).
9. [Delete the CiLogon client](#delete-the-cilogon-client)

## Delete the alert policy

Go back to the GCP console, and under Monitoring -> Alerting, click on the
deployment's Policy and then click on Delete
If you don't disable the alerts, a page will be sent off when GCP is unable to
reach the `prod` deployment of the hub you're removing.

Open up the [GCP console](https://console.cloud.google.com/) and using the
sidebar, navigate to Monitoring -> Alerting.  Search for the deployment's
Policy, click on the link, and then click on "Delete".

## Delete the Helm deployments

Ensure you're logged in to GCP on the command line, and run the following two
commands (replace `<hubname>` with the hub name):

``` bash
helm delete -n <hubname>-prod <hubname>-prod
helm delete -n <hubname>-staging <hubname>-staging
```

This effectively deletes the hub's kubernetes deployments.

## Archive or delete NFS storage

This step depends on what the institution want to do, if anything, with the
user homedirs on the NFS server.

If there is some user data, then it would be best to archive it for a certain
amount of time (probably a year at minimum).  If the hub has never been used,
or only a couple of instructors had logged in to "kick the tires", then the
NFS directories should be fine to delete completely.

To archive the deployment's NFS directories, run the following commands:

``` bash
pod_name=$(kubectl get pod -n jupyterhub-home-nfs -l app.kubernetes.io/component=nfs-server -o "jsonpath={.items[0].metadata.name}")
kubectl exec -n jupyterhub-home-nfs ${pod_name} -- sh -c "tar -zcvf /export/<hubname>.tar.gz /export/<hubname> && ls -l /export/<hubname>.tar.gz"
```

Be sure that the deployment's homedir archive has been created before deleting
anything.

Then, you can delete the directory by running:

``` bash
kubectl exec -n jupyterhub-home-nfs ${pod_name} -- sh -c "rm -rf/export/<hubname>"
```

## Remove deployment from labeler action

Create a new feature branch from `staging` in your local clone of
`cal-icor-hubs` before continuing:

``` bash
github checkout -b remove-<hubname>
```

Edit `.github/labeler.yml` and remove the hub's entry located towards the end
of this file.

## Remove deployment from hub repo

Next, delete the folder under `deployments/` for this hub:

``` bash
git rm -rf deployments/<hubname>
```

## Update GitHub

Next, we will remove the GitHub labels and the URLs in the GitHub Issue
template folder.

``` bash
gh label delete "hub: <hubname>"
```

Edit the follow files found in the `.github/ISSUE_TEMPLATE/` folder and remove
the hub's URL from each one:

``` bash
additional_storage_request.yaml
admin_request.yaml
cpu_template.yml
memory_request.yml
package_request.yml
```

## Review your changes

Run `git diff` and ensure everything looks good!  After that, add/commit and
push to the `cal-icor-hubs` repository and create a PR.

## Review and merge

Ensure that just the expected files are going to be removed from the repository
and that the hub's labels aren't added to the PR.

Once you're happy that things look good, merge to `staging`.  This can be
merged to prod at your leisure.

## Delete the CiLogon client

Run the following command to delete the deployment's CiLogon client:

``` bash
./scripts/cilogon_clients.py remove <hubname>
```
