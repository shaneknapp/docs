# Create a New Single User Image

A user image is the environment a student lands in: the packages, the kernels, the
RStudio install if there is one. Each one lives in its own repo under
[cal-icor](https://github.com/cal-icor/), gets built by
[repo2docker](https://github.com/jupyterhub/repo2docker), and is pushed to the
`user-images` Artifact Registry repository.

The shared
[base-user-image](https://github.com/cal-icor/base-user-image) is typically
sufficient for most hubs.

Create a new image when a course needs software the base image does not have,
or needs a version of something the other hubs cannot move to.

Building is all CI. Merge a change, and Actions builds the image, pushes it, and
opens a PR against `cal-icor-hubs` bumping the tag. Merging that PR deploys the
hub(s) that use the image.

## The short version

1. [Create the repo](#create-the-repository) from the `base-user-image` template.
2. [Set `IMAGE`, and `HUB` if the image backs a hub](#repository-settings).
3. [Turn off Actions on your fork](#disable-actions-on-your-fork).
4. [Point the hub's config at the image](#point-a-hub-at-the-image) with a
   `PLACEHOLDER` tag.
5. [Register the repo](#register-the-repo) in `configs/image-repos.txt` and Slack.
6. [Edit the image](#edit-the-image), test it locally, PR it.
7. [Merge to `main`](#the-build-loop) and merge the PR the build opens.

## Create the repository

[base-user-image](https://github.com/cal-icor/base-user-image) is a template repo.
Use this template, then Create a new repository. The owner has to be `cal-icor`,
and the name has to end in `-user-image`, since the repo name is also the image
name. Fork it afterwards and work from the fork.

The template ships the pieces repo2docker reads (`environment.yml`, `apt.txt`,
`runtime.txt`, `postBuild`, `start`), both CI workflows, and the two test suites.

If the image needs software installed as `root`, add a
[Dockerfile.appendix](https://repo2docker.readthedocs.io/en/latest/usage.html#cmdoption-jupyter-repo2docker-appendix).
For something genuinely complicated, a plain
[Dockerfile](https://github.com/cal-icor/csumb-user-image/tree/main)
works too, but prefer repo2docker.

To base the image on an existing one rather than `base-user-image`, still create
the repo from the template, then copy the other image's files in. That way the
workflows and settings stay right.

## Repository settings

Under Settings, General, Pull Requests, check Automatically delete head branches.

Under Settings, Secrets and variables, Actions, Variables tab, add:

| Variable | Value | Required |
|---|---|---|
| `IMAGE` | `cal-icor-hubs/user-images/<repo name>` | always |
| `HUB` | the hub name, for example `csumb` | only if a hub deploys this image |

`IMAGE` has no registry hostname in it. The workflow supplies
`us-central1-docker.pkg.dev`, and the same string is what it greps for in the hub
configs, so it has to match what `common.yaml` says after the hostname.

`HUB` is the deployment name the auto-PR's branch and title use. Leave it unset for
an image nothing deploys directly, like
[base-python-image](https://github.com/cal-icor/base-python-image); the build then
stops after pushing to the registry.

### Disable Actions on your fork

Settings, Actions, General, Disable actions, Save.

Otherwise every push to your fork runs the workflows there. They lack the
permissions to finish, so they fail and GitHub nags you about it each time.

## Point a hub at the image

The hub has to exist first. If it does not, please
[create a new hub](new_hub) before this.

In `deployments/<hubname>/config/common.yaml`, set the image name and put
`PLACEHOLDER` where the tag goes:

```yaml
jupyterhub:
  # a bunch of hub config here...
  # ...
  singleuser:
    image:
      name: us-central1-docker.pkg.dev/cal-icor-hubs/user-images/<hubname>-user-image
      tag: PLACEHOLDER
```

The first build replaces `PLACEHOLDER` with the image SHA. `staging.yaml` and
`prod.yaml` can carry their own image block to pin one environment to a specific
tag, and they override `common.yaml`.

Merge that to `staging` with the `hub: <name>` label stripped. The image does not
exist yet, so a deploy would fail on the `PLACEHOLDER` tag. With no hub label the
hub layer resolves an empty list and skips. See
[the deployment pipeline](deploy_pipeline).

## Register the repo

Add the new repo to
[`configs/image-repos.txt`](https://github.com/cal-icor/cal-icor-hubs/blob/staging/configs/image-repos.txt)
in `cal-icor-hubs`. It is the canonical list of image repos, and what fleet-wide
sweeps clone from. Commented-out lines are images no longer in use.

Then subscribe the channel to it. In `#cal-icor-bots`:

``` bash
/github subscribe cal-icor/<repo name>
```

## Edit the image

Create a feature branch and change whatever the course needs. `environment.yml`
holds the conda packages and is where most of the work happens. Pin versions if
possible.

Copy `README-template.md` over `README.md` and replace `<IMAGE-NAME>` throughout.

Build it locally before you push. CI gives a build 90 minutes; a broken
`environment.yml` fails locally in a fraction of that.

Full instructions, including the extra arguments Apple silicon needs, are in
[testing repo2docker locally](repo2docker_local.md) and in the image repo's own
README.

Two optional test directories, both shipped by the template and both run by CI when
present:

- `image-tests/` holds pytest notebook and R tests, run against the built image.
- `browser-tests/` starts the image as a JupyterLab server on port 8888 and drives
  its REST and WebSocket API with Playwright.

## The build loop

Two workflows, both thin callers of
[shared-workflows](https://github.com/cal-icor/shared-workflows). They pin a tag,
not a branch, so bumping the shared workflow is a deliberate edit.

`build-test-image.yaml` runs on every PR. It builds and tests, but never pushes, so
a broken `environment.yml` fails here rather than in the registry.

`build-push-create-pr.yaml` runs on merge to `main`. It builds, pushes to the
registry tagged with the commit SHA, runs the browser tests against what it just
built, then rewrites the tag in every `common.yaml` in `cal-icor-hubs` that
references this image and opens a PR against `staging`. The PR comes from
`cal-icor-create-pr-bot` on a branch named `update-<hub>-image-tag-<sha>`, and its
body links back to the image PR that triggered it.

Neither runs for changes to the READMEs, `LICENSE`, `.github/**` or the `images/`
screenshots. The push build also skips the two test directories; the PR build does
not, so a change to a test still gets tested.

The loop:

1. PR your branch against `main`. Wait for the test build.
2. Merge it. Watch the build in the repo's Actions tab.
3. Find the PR it opened in
   [cal-icor-hubs](https://github.com/cal-icor/cal-icor-hubs/pulls), check the diff
   is only tag changes, and merge it to `staging`.
4. Watch
   [the deploy](https://github.com/cal-icor/cal-icor-hubs/actions/workflows/deploy-spring-2025.yaml).

:::{admonition} A shared image deploys every hub that uses it
:class: warning
The tag update is not scoped to one hub. It rewrites every `common.yaml` that
references the image, so the PR picks up a hub label for each one and merging
deploys all of them at once. A recent `base-user-image` bump touched 16 files and
carried 15 hub labels. Check the label list on the PR before merging.
:::
