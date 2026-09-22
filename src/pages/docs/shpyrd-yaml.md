---
title: shpyrd.yaml
description: The project file - which app a repository is, its process types, sizes, build settings and domains.
---

`shpyrd.yaml` lives at the root of the directory you deploy from and plays the role of `fly.toml` or a `Procfile` plus `app.json`. Everything but `app` is optional. {% .lead %}

```yaml
# Which project this repository (or directory) deploys to.
project: hello-world

# Process types. Declared types are authoritative: a type removed here is
# removed from the cluster on the next deploy. Instance counts set with
# `shpyrd scale` survive unless pinned with `replicas`.
processes:
  web:
    port: 8080          # exposed through the URL; PORT is injected. Default 8080 for web.
    size: shared-m      # instance size from the cluster catalog (shpyrd sizes list). Default: shared-s.
    volumes:
      - name: data      # a volume of the project (shpyrd volumes create data --size 5Gi)
        path: /data
  worker:
    size: shared-xs
    replicas: 2         # pin the instance count
    command: ["/cnb/process/worker"]   # override the entrypoint (required for non-web types of Dockerfile images)
    args: ["--queue", "default"]

# Build settings.
build:
  strategy: buildpacks  # or dockerfile; without it, dockerfile when the directory has a Dockerfile
  env:
    BP_GO_TARGETS: ./cmd/web:./cmd/worker   # buildpacks: any BP_* variable of the buildpack in use
    BP_NODE_VERSION: "22.*"                 # Dockerfile: build arguments
  builder: shpyrd       # kpack ClusterBuilder; the default is fine
  dockerfile: Dockerfile  # dockerfile strategy: path inside the deployed directory
  target: runtime         # dockerfile strategy: multi-stage target

# Extra hostnames for the web process (the default <app>.<domain> stays).
domains:
  - hello.example.test
```

## Fields

| Field | Meaning |
| --- | --- |
| `project` | Project name; used when `--project` is not given. Created with `shpyrd projects create <name>` (`--save` writes this file). `app` is accepted as an alias. |
| `processes.<type>.port` | Port the process listens on. `web` defaults to 8080 and is published through the URL; other types get no port unless set. `PORT` is injected. |
| `processes.<type>.replicas` | Pin the number of instances. Without it, `shpyrd scale` values are kept across deploys (default 1). |
| `processes.<type>.size` | Instance size from the cluster catalog (`shpyrd sizes list`): `shared-*` sizes get a guaranteed CPU share that can burst up to 4×, `dedicated-*` sizes get whole cores. Default: the catalog default (`shared-s`, 0.5 CPU / 64 MiB out of the box). Changing it is a release. |
| `processes.<type>.cpu`, `memory` | Override the size's limits (e.g. `cpu: "1"`, `memory: 1Gi`). Prefer a size; use these for one-off needs. |
| `processes.<type>.command`, `args` | Override the command. By default `web` runs the image entrypoint and other types run `/cnb/process/<type>`, the process the buildpacks recorded under that name. Dockerfile images have a single entrypoint, so every type other than `web` must set `command`. |
| `processes.<type>.volumes` | Project volumes to mount: `name` (created with `shpyrd volumes create`) and `path`. A single-instance volume pins the process to one instance; see [Resources](/docs/resources#volumes). |
| `build.strategy` | `buildpacks` (default) or `dockerfile`. Without it, `shpyrd deploy` picks `dockerfile` when the deployed directory has a `Dockerfile`. |
| `build.env` | Environment for the build: buildpack configuration such as `BP_GO_TARGETS`, `BP_JVM_VERSION`, `BP_NODE_RUN_SCRIPTS`, or `ARG` values for a Dockerfile. Runtime config vars are set with `shpyrd secrets`, not here. |
| `build.builder` | kpack `ClusterBuilder` to use (buildpacks). |
| `build.dockerfile`, `build.target` | Dockerfile path relative to the deployed directory (default `Dockerfile`) and the multi-stage target to build. |
| `domains` | Additional hostnames; certificates are issued for each. |

## Process types and buildpacks

The buildpacks decide which process types an image has:

- **Go**: one per build target; use `BP_GO_TARGETS=./cmd/a:./cmd/b` to build several. The first is the default (`web`).
- **Node.js**: `web` from `npm start`/`package.json`; other types via a `Procfile`.
- **Java, Python, Ruby, .NET**: the framework's entry point becomes `web`; add a `Procfile` for others.
- **Procfile**: `worker: bundle exec sidekiq` adds a `worker` type on any stack.

`processes` in `shpyrd.yaml` tells shpyrd which of those types to run and how many instances; the names must match what the image provides.

With a Dockerfile there is no such catalogue: `web` runs the image's `CMD`, and each other type names its `command` (`worker: { command: ["node", "worker.js"] }`).
