---
title: Deploying
description: Create a project, deploy it from a checkout or a Git URL, configure it, scale it, read its logs and roll back.
---

The CLI talks to your cluster with your kubeconfig; the only call that reaches the shpyrd server is the upload of your source archive. {% .lead %}

## Create a project

```shell
shpyrd projects create my-service          # namespace app-my-service + App resource
shpyrd projects create my-service --save   # also writes shpyrd.yaml (app: my-service) in the current directory
```

Names are lowercase letters, digits and dashes (max 40 characters) and become the hostname: `https://my-service.<domain>`.

## Deploy

From inside your repository:

```shell
shpyrd deploy                 # app from shpyrd.yaml, or --project my-service
```

What happens:

1. **Archive.** The committed tree of the current directory (`git archive HEAD`) is packed; run it from a subdirectory to deploy just that service of a monorepo. Uncommitted changes are not included unless you pass `--working-tree` (also chosen automatically when nothing in the directory is committed yet). Outside a Git repository the directory is tarred.
2. **Upload.** The archive is sent to the cluster through the Kubernetes API server (no ingress or token needed).
3. **Build.** In the cluster, with the Paketo buildpacks (kpack) or, when the directory has a `Dockerfile`, with BuildKit; the CLI streams every step.
4. **Release.** The controller rolls the new image out process by process and prints the release number and URL.

```
==> Archiving HEAD:examples/hello (654f4925638e)
==> Uploading source (2.6 KiB)
==> Building
===> prepare
===> detect
4 of 9 buildpacks participating
===> build
    web (default): /layers/paketo-buildpacks_go-build/targets/bin/web
    worker:        /layers/paketo-buildpacks_go-build/targets/bin/worker
===> export
==> Releasing
    Deploying: Releasing v3: web 1/3 updated · worker 0/1 updated
    Running: web 3/3 · worker 1/1
Released v3: Deploy 654f4925638e
https://hello-world.127.0.0.1.nip.io
```

Other sources:

```shell
shpyrd deploy --git https://github.com/org/repo --ref main --path services/api   # new commits rebuild automatically
shpyrd deploy --image ghcr.io/org/repo:1.4.2                                   # run a prebuilt image, no build
shpyrd deploy --no-wait                                                         # do not follow the build
```

Deploying from Git is also available in the dashboard (**Deploy** button, or when creating the project).

{% callout title="Which languages?" %}
Anything the Paketo buildpacks understand: Go, Node.js, Java, Python, Ruby, .NET Core and static sites served by nginx/httpd. Repositories with a `Dockerfile` are built with BuildKit instead (below), and `--image` runs anything already built.
{% /callout %}

## Dockerfile builds

When the deployed directory contains a `Dockerfile`, `shpyrd deploy` builds it instead of using buildpacks:

```
==> Archiving working tree (c26f84642501)
==> Building with Dockerfile (Dockerfile)
==> Uploading source (1.7 KiB)
==> Building
===> fetch
===> build
#8 importing cache manifest from 10.96.0.50:5000/apps/hello-docker:cache
#9 CACHED
...
pushed 10.96.0.50:5000/apps/hello-docker@sha256:2a6749dc...
==> Releasing
Released v2: Deploy c26f84642501
```

The build runs as a rootless [BuildKit](https://github.com/moby/buildkit) Job in the project namespace: a first step fetches your archive (or clones the Git revision), the second builds the context and pushes the image. Multi-stage builds, build arguments, `.dockerignore` and a layer cache between builds all work as with `docker build`.

Pin or tune it in [`shpyrd.yaml`](/docs/shpyrd-yaml):

```yaml
build:
  strategy: dockerfile        # buildpacks | dockerfile; without it, auto-detected from the Dockerfile
  dockerfile: deploy/Dockerfile
  target: runtime             # multi-stage target
  env:
    NODE_ENV: production      # build arguments
processes:
  web: {}                     # runs the image CMD
  worker:
    command: ["node", "worker.js"]   # Dockerfile images have one entrypoint: other process types name their command
```

From Git, detection is not possible; say so explicitly: `shpyrd deploy --git https://github.com/o/r --dockerfile` (optionally `--dockerfile deploy/Dockerfile`). The dashboard's **Deploy** dialog has the same choice. Git sources with a Dockerfile are rebuilt when the revision or the build settings change, not on every new commit as buildpack builds are; pass a commit or redeploy to rebuild a branch.

Failures show BuildKit's error in the CLI, the Activity panel and `shpyrd projects info`; the previous release keeps serving. `examples/hello-docker` in the repository is a complete example.

## Processes and sizes

Declare process types in [`shpyrd.yaml`](/docs/shpyrd-yaml) next to your code; `shpyrd deploy` applies it:

```yaml
project: hello-world
processes:
  web:
    port: 8080
  worker:
    cpu: "500m"
    memory: 256Mi
build:
  env:
    BP_GO_TARGETS: ./cmd/web:./cmd/worker   # Go: one process per command
```

Scale at any time; counts survive deploys:

```shell
shpyrd scale web=3 worker=2
```

## Config vars

```shell
shpyrd secrets set DATABASE_URL=postgres://... LOG_LEVEL=debug
shpyrd secrets unset LOG_LEVEL
shpyrd secrets list            # names and when each was last set; values are never shown
```

Every change is a release (`Set DATABASE_URL config var`) and restarts the processes with the new environment. The dashboard's **Config** tab does the same, including pasting `.env` files.

## Shell and one-off commands

```shell
shpyrd shell                          # bash (or sh) in web.1
shpyrd shell --instance worker.2      # a specific instance
shpyrd shell -- cat /etc/os-release   # run one command and return its exit code
```

`shpyrd shell` attaches to a **running instance**: what you see is the live process's filesystem and environment. Buildpack images get the same environment as the process (through the CNB launcher), so `node`, `bundle` or `python` are on the `PATH`.

```shell
shpyrd run rails db:migrate                 # a new instance of the current release, removed when the command exits
shpyrd run --size shared-l python manage.py import big.csv
shpyrd run --detach ./nightly.sh            # start and return; follow with shpyrd logs -p run
```

`shpyrd run` starts a **temporary instance** (like `heroku run`) with the release's image and config vars, streams its output and exits with the command's exit code; piped input works (`cat dump.sql | shpyrd run psql`). Instances left by `--detach` or a killed terminal are cleaned up after they finish.

## Volumes

Processes that need a disk mount a project volume:

```shell
shpyrd volumes create data --size 5Gi        # a persistent disk of the project
```

```yaml
# shpyrd.yaml
processes:
  web:
    volumes:
      - name: data
        path: /data
```

Volumes are persistent: they outlive deploys, scaling and crashes and are deleted only by `shpyrd volumes delete` (or the project's destruction). A volume is **single-instance** by default (block storage attaches to one node): the process mounting it runs one instance and rolls out with a stop-then-start (a few seconds of downtime per deploy, no data risk), and `shpyrd scale web=3` is refused with that explanation. `--shared` volumes can be mounted by many instances and processes but need a provisioner that offers `ReadWriteMany`; they are unsafe for SQLite. See [Resources](/docs/resources) for the details.

## Logs

```shell
shpyrd logs -f                       # all instances, live
shpyrd logs --process worker -n 200  # one process type, last 200 lines per instance
shpyrd logs --build                  # the latest build's output
shpyrd logs -p run                   # one-off instances started with shpyrd run --detach
```

```
2026-09-22T00:54:24 web.1    | GET / from 10.244.0.5:58732 (request #2)
2026-09-22T00:54:24 worker.1 | job #3 done in 200ms: "Olá mundo"
```

## Releases and rollback

```shell
shpyrd releases
```

```
RELEASE       CREATED              DIGEST        DESCRIPTION
v7 (current)  2026-09-21 22:44:30  2edf5661353b  Rollback to v5
v6            2026-09-21 22:44:18  2edf5661353b  Set GREETING config var
v5            2026-09-21 22:44:06  2edf5661353b  Set GREETING config var
v4            2026-09-21 21:53:52  2edf5661353b  Deploy 654f4925638e
```

```shell
shpyrd rollback        # to the release before the current one
shpyrd rollback 4      # to a specific release: its build and its config vars
```

A rollback is refused while another release is still rolling out (`--force` overrides). See [Concepts](/docs/concepts#releases) for what a release contains.

## Open and inspect

```shell
shpyrd open                 # opens https://my-service.<domain>
shpyrd projects info my-service     # status, releases and every resource of the project (app, volumes...)
shpyrd projects list
kubectl -n app-my-service get all,ingress,volumes.shpyrd.io,image.kpack.io,jobs
```

## Destroy

```shell
shpyrd projects destroy my-service      # deletes namespace app-my-service and everything in it
```

The command warns about the data on the project's volumes before asking for confirmation. Images stay in the registry.
