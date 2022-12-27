# Let's code a reusable workflow for building state-of-the-art, multi-platform Docker images with GitHub Actions ✨

I wrote a reusable GitHub Action workflow for one of my pet projects, [rickroller](https://github.com/derlin/rickroller). The workflow itself is very small and uses mostly well-known Actions, but encompasses a lot of complex? ideas and logic.

So let's go together through all the steps required to create a robust and reusable workflow to build state-of-the-art, multi-architecture images with GitHub Actions.

* * *

The full workflow is available here: https://github.com/derlin/rickroller/blob/main/.github/workflows/reusable\_docker-build-and-push.yaml. Use [this link](https://github.com/derlin/rickroller/blob/8b03ab644c8888d4bd04648977d1c1161f1cc66a/.github/workflows/reusable_docker-build-and-push.yaml) to consult the exact version at the time of writing.

As [rickroller](https://github.com/derlin/rickroller) is a pretext to play with Google Cloud Run, GitHub actions and to try Open Source best practices, you can find a lot of details in the [README](https://github.com/derlin/rickroller) about GitHub Actions, but not only. Feel free to have a look and leave a ⭐ !

* * *

*NOTE*: I assume you have a basic knowledge of GitHub Actions and Docker.

## The context and goal

Being in 2022, my python project is deployed using a Docker image. This image is built and (pushed to a registry) in multiple "processes":

*   during a PR - so I can test my container (tag like `pr-4`),
    
*   when there is a new push to main - so people can preview new features (pulling the image `latest` or `main-eeffaa2`),
    
*   and from a release (with tag `v1.0.0`).
    

It thus makes sense to create one reusable workflow, that can be called in different CI contexts.

Having a Mac M1 myself, it is important users with AMD or ARM processors can run rickroller. It must be available for multiple architectures.

Speed and security are also important topics: the Dockerfile should be scanned for vulnerabilities, and the workflow should avoid building the same layer over and over when it can actually just build and cache it once.

To sum it up, I want:

1.  a reusable workflow,
    
2.  that builds a multi-arch image (at least AMD64 and ARM64),
    
3.  with caching to speed up the process,
    
4.  running some security scan,
    
5.  able to push to GitHub Registry,
    
6.  with meaningful image tags (`latest`, `pr-xxxx`, `main-ef34221` `vX.X.X`) and [annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md)
    

Let's get started!

## The basics: a reusable workflow

GitHub introduced [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) and [composite actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) about a year ago as an attempt to enhance reusability and DRY-ness (**D**on't **R**epeat **Y**ourself).

In short, **composite actions** let you create actions that are calling other actions. They are still actions, that need to be called from within a series of steps.

A **reusable workflow** - or *callable workflow* is instead a complete *job* (with checkout, etc), that runs in a specific runner. It is less flexible, as you cannot add a step before or after it (you can, however, define another job that generates its inputs or depends on its output).

The documentation is very well written, so I won't go too much into the details. In short, to create a reusable workflow, you simply set its triggers to `on: workflow_call`. You can then define inputs, secrets, etc. that can be later referenced using `${{ inputs.some_input }}` or `${{ secrets.some_secret }}`:

```yaml
name: Reusable Workflow Example
on:
  workflow_call:
    inputs:
      some_input:
        description: Just an example input
        type: boolean
        required: false
        default: false
    secrets:
      #...

jobs:
  my_job:
    name: Do Something
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3 
      # ...
```

For my reusable workflow, I have 2 inputs:

1.  `publish` (boolean): whether to publish the image (to GitHub Registry),
    
2.  `version` (string): the version being released (empty if not called from a release workflow).
    

## The steps

The GitHub community has all the Actions I need already made up to perform the painful tasks of setting up the environment, building, and pushing. The only difficulty is finding out which ones to use, and how to configure them.

### Extracting Labels and Tags

No need to manually and painfully determine the proper labels and tags, as an Action is already available for this task: [docker/metadata-action](https://github.com/docker/metadata-action) - `v3` at the time of writing.

#### Labels

Docker image **labels** are a way to attach (any) key-value metadata to an image. These metadata are not available to the running container but are rather used to share information like where the source code for the image resides, who supports it, or what CI build/codebase version generated it.

It is common to use the OCI - Open Container Initiative - [set of standardized labels](https://github.com/opencontainers/image-spec/blob/main/annotations.md), all prefixed with `org.opencontainers.image.*`. The most common being `.source`, `.version`, and `.revision`.

The `docker/metadata-action` automatically extracts OCI Labels based on the commit and the repository's metadata. Here is what it computes for my rickroller repository:

```json
{
    "org.opencontainers.image.created": "2022-04-15T06:13:02.269Z",
    "org.opencontainers.image.description": "A simple Python app to test GitHub CI",
    "org.opencontainers.image.licenses": "",
    "org.opencontainers.image.revision": "83592e8567ee6bfb01caccae5e791ab38b5ee7e0",
    "org.opencontainers.image.source": "https://github.com/derlin/rickroller",
    "org.opencontainers.image.title": "rickroller",
    "org.opencontainers.image.url": "https://github.com/derlin/rickroller",
    "org.opencontainers.image.version": "main-83592e8"
}
```

#### Tags

Before building, we need to determine the ID(s) of the images, for example, `ghcr.io/derlin/rickroller:latest`. There are three parts: `ghcr.io`, which matches the name of the registry you push the image to, `derlin/rickroller`, the user+ID of your project (usually static), and finally the *tag* or *version*, in this case, `latest`.

Ideally, tags should be meaningful. Images from PRs should be versioned following e.g. the format `pr-{id}`, images from branch main something like `main-{sha (short)}`, and releases `vX.X.X`. The version `latest` is known as a *moving tag*, as it points to a different image (usually from the latest successful build on the main branch) as time goes on.

That's a lot of combinations and ifs... Fortunately, [docker/metadata-action](https://github.com/docker/metadata-action) can do everything, provided the right set of inputs.

Here is my configuration for rickroller:

```yaml
- name: Extract Docker metadata
  id: meta
  uses: docker/metadata-action@v3
  with:
    images: ghcr.io/${{ github.repository }}
    tags: |
      type=semver,pattern={{version}},value=${{ inputs.version }}
      type=semver,pattern={{major}}.{{minor}},value=${{ inputs.version }}
      type=semver,pattern={{major}},value=${{ inputs.version }}
      type=ref,event=branch,suffix=-{{ sha }}
      type=ref,event=pr
      type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/') }}
    flavor: |
      latest=false
```

Let's break it down. First, I give an `id` to the step, so I can refer to its output later in the workflow.

Next, I provide a list of image names, without the tag. Since I only push to GitHub Registry (ghcr.io), I can use `ghcr.io/${{ github.repository }}` which will be resolved as `ghcr.io/derlin/rickroller`.

The magic is in the `tags` input. The Action supports a list of "conditions" that will be evaluated to generate the right tags depending on things like branches, triggers, etc.

In this specific case, the first lines starting with `type=semver` are used when a `version` is passed to the workflow (e.g. from a release). The version input must have the form `X.Y.Z` - or more accurately `{major}.{minor}.{patch}` (semantic versioning). With those 3 lines, the Action returns the Docker tags `X.Y.Z`, `X.Y`, and `X` when the `value=` isn't evaluated to an empty string (i.e. when the `version` input is provided).

Next, there are the `type=ref,event={event}` lines to conditionally get tags based on the trigger of the workflow (`event=`). If this is from a `pr` the Action generates the tag `pr-{number}`. If it is from a branch, it generates `{branch_name}`. Since I want to have the branch name suffixed with a unique id, I also specify `suffix=-{{ sha }}`. The `{{ sha }}` will be evaluated by the Action to the short SHA of the git commit (`git rev-parse --short`), and appended to the branch name (for example `main-eeffaa3`).

Finally, to have the `latest` tag set only for the `main` branch and a release, I disable the default "latest flavor" (which always adds the tag `latest`) and provide instead the expression:

```yaml
type=raw,value=latest,enable=${{ github.ref == 
  'refs/heads/main' || startsWith(github.ref, 'refs/tags/') }}
```

This line says "*use the raw value "latest", but only if the condition* `enable=` evaluates to true".

As you can see, it is quite powerful!

#### Output

The metadata Action returns the metadata and tags in multiple formats, but what is important is that I can later reference them using:

```yaml
# meta is the id we gave to the metadata action
${{ steps.meta.outputs.tags }}
${{ steps.meta.outputs.labels }}
```

### Scanning the Dockerfile for vulnerabilities

In production settings, the full Docker images should be scanned with state-of-the-art tools such as [sysdig](https://sysdig.com/use-cases/vulnerability-management/), [Docker scan](https://docs.docker.com/engine/scan/), [snyk](https://snyk.io/learn/docker-security-scanning/), or the like. Those tools scan the built image itself and are thus able to detect vulnerabilities not only in your Dockerfile, but in all its above layers, as well as in the softwares your Dockerfile brings in.

There are plenty to choose from, but none of them is completely free. You always need an account, a specific setup, and are often limited in the number of scans or repositories.

So in this workflow, I chose to limit myself to a "linter", checkov. Feel free to augment this example with a real scan!

> [Checkov](https://checkov.io) *supports the evaluation of policies on your Dockerfile files*. \[...\] *it will validate if the file is compliant with Docker best practices such as not using root user, making sure health check exists and not exposing SSH port*.

The full list of Dockerfile policies it checks can be found [here](https://www.checkov.io/5.Policy%20Index/dockerfile.html).

```yaml
- name: Lint Dockerfile using Checkov
  id: checkov
  uses: bridgecrewio/checkov-action@master
  with:
    directory: .
    framework: dockerfile # only ask for dockerfile scans
    quiet: true # show only failed checks
    container_user: 1000 # UID to run the container under
     # to prevent permission issues
```

Checkov will parse the Dockerfile, and report the results:

```yaml
Passed checks: 9, Failed checks: 0, Skipped checks: 0

...
Check: CKV_DOCKER_3: "Ensure that a user for the container has been created"
        PASSED for resource: Dockerfile.USER
        File: Dockerfile:49-49
        Guide: https://docs.bridgecrew.io/docs/ensure-that-a-user-for-the-container-has-been-created
...
```

If you have python installed, it is possible to run the same check locally:

```bash
pip install checkov
checkov --framework dockerfile -f Dockerfile
```

### Building and pushing a multi-arch image

#### Logging into the registry

To be able to push to a registry, I need to log in first. By default, the workflow inherits the `GITHUB_TOKEN`, so logging to ghcr.io is as easy as calling:

```yaml
- name: Login to Container Registry
  uses: docker/login-action@v2
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

#### Setting up QEMU and buildx

[QEMU](https://www.qemu.org) lets you "*run operating systems for any machine, on any supported architecture*", while [buildx](https://github.com/docker/buildx) is "*a Docker CLI plugin for extended build capabilities with* [*BuildKit*](https://github.com/moby/buildkit)". Both are necessary to build Docker images targeting another platform/architecture.

Setting them up is again available through simple Actions:

```yaml
- name: Set up QEMU
  uses: docker/setup-qemu-action@v2

- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v2
```

To learn how to use buildx locally to build multi-platform images, have a look at the [docs](https://docs.docker.com/build/building/multi-platform/) and docker.com's blog post: [How to Rapidly Build Multi-Architecture Images with Buildx](https://www.docker.com/blog/how-to-rapidly-build-multi-architecture-images-with-buildx/).

#### Building and pushing the image (finally)

We finally have all the bricks in place to build and push a multi-arch Docker image using the [docker/build-push-action](https://github.com/docker/build-push-action):

```yaml
- name: Build and push Docker image
  uses: docker/build-push-action@v3
  with:
    # the Dockerfile is at the root of the workspace
    context: .
    # Build for AMD and ARM (requires buildx+qemu)
    platforms: linux/amd64,linux/arm64
    # only push when requested
    push: ${{ inputs.publish }}
    # pass the output of the metadata action
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    # use layer caching
    # The mode=max is to also cache the builder image
    # (vs only the final image - mode: min)
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

(for `cache-from` and `cache-to`, keep reading !)

#### Layer caching

The only thing I haven't talked about is **Docker layer caching (DLC)**, which is a great feature when building Docker images as a regular part of the CI process. The idea is to cache the individual layers of Docker images built in CI jobs, and then reuse unchanged image layers on subsequent runs, rather than rebuilding the entire image from scratch every time.

This caching mechanism is a given when building Docker images locally (see Docker's documentation - [leverage build cache](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache)). However, in CI, a new runner is started each time, so the cache is always empty.

The build-push-action from Docker [supports multiple types of caches](https://github.com/docker/build-push-action/blob/master/docs/advanced/cache.md). In this workflow (see precedent section), I use the *GitHub cache* (`gha`). It is rather straightforward to turn on. Simply set the `cache-from` and `cache-to` parameters:

```yaml
- uses: docker/build-push-action@v3
  with:
    # ...
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

One important detail is the `mode=max`, which instructs the Action to cache **all** layers, and not only the ones from the final image. It is very important if the Dockerfile is using [multi-stage builds](https://docs.docker.com/build/building/multi-stage/): without it, the layers from the builder image are ignored.

At the time of writing, GHA is limited to 10G, and isn't shared between branches.

Quick reminder: if a top layer changes, all subsequent layers will change as well, so devise your Dockerfiles wisely!

## Calling the workflow

To call this reusable workflow from the same repository, I simply use:

```yaml
name: ...

jobs:
  # ... other jobs ? ...
  docker:
    uses: ./.github/workflows/reusable_docker-build-and-push.yaml
    with:
      publish: true
      # ... other inputs ...
```

it is also possible to call it from another repository, in which case the repository path must be provided:

```yaml
uses: derlin/rickroller/.github/workflows/reusable_docker-build-and-push.yaml
```

## We did it !

Wow. What a journey.

The full workflow as well as usages are available in my repo: {% embed https://github.com/derlin/rickroller %}.

I hope you learned something, happy coding ✨

* * *

## BONUS: *optionally* publish the Image to Docker Hub

As a bonus, I wanted to be able to also push to Docker Hub, but only in specific situations (e.g. from a release, but not from a PR build) and with a different set of tags (no `main-{sha}`).

This conditionality was a bummer, as it is not supported by default. After playing around a bit, I designed a hack to implement this "if" without duplicating the whole workflow.

If you are interested, have a look at the updated workflow: [https://github.com/derlin/rickroller/blob/e67546a90218c8300676c1270699cccdb5f7e053/.github/workflows/reusable\_docker-build-and-push.yaml](https://github.com/derlin/rickroller/blob/e67546a90218c8300676c1270699cccdb5f7e053/.github/workflows/reusable_docker-build-and-push.yaml) and let me know in the comments if you would like a post about it!