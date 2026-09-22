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
3. **Build.** kpack builds it with the Paketo buildpacks; the CLI streams every step.
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
Anything the Paketo buildpacks understand: Go, Node.js, Java, Python, Ruby, .NET Core and static sites served by nginx/httpd. Dockerfiles are not used yet; `--image` covers prebuilt images meanwhile.
{% /callout %}

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

## Logs

```shell
shpyrd logs -f                       # all instances, live
shpyrd logs --process worker -n 200  # one process type, last 200 lines per instance
shpyrd logs --build                  # the latest build's output
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
shpyrd projects info my-service
shpyrd projects list
kubectl -n app-my-service get all,ingress,image.kpack.io
```

## Destroy

```shell
shpyrd projects destroy my-service      # deletes namespace app-my-service and everything in it
```

Images stay in the registry.
