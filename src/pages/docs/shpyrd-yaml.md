---
title: shpyrd.yaml
description: The project file - which app a repository is, its process types, sizes, build settings and domains.
---

`shpyrd.yaml` lives at the root of the directory you deploy from and plays the role of `fly.toml` or a `Procfile` plus `app.json`. Everything but `app` is optional. {% .lead %}

```yaml
# Which project this repository (or directory) deploys to.
app: hello-world

# Process types. Declared types are authoritative: a type removed here is
# removed from the cluster on the next deploy. Instance counts set with
# `shpyrd scale` survive unless pinned with `replicas`.
processes:
  web:
    port: 8080          # exposed through the URL; PORT is injected. Default 8080 for web.
    cpu: "1"            # size (limit). Default 1 CPU.
    memory: 512Mi       # size (limit). Default 512Mi.
  worker:
    replicas: 2         # pin the instance count
    command: ["/cnb/process/worker"]   # override the entrypoint (rarely needed)
    args: ["--queue", "default"]

# Build-time settings passed to the buildpacks.
build:
  env:
    BP_GO_TARGETS: ./cmd/web:./cmd/worker   # Go: build several commands, one process type each
    BP_NODE_VERSION: "22.*"                 # any BP_* variable of the buildpack in use
  builder: shpyrd       # kpack ClusterBuilder; the default is fine

# Extra hostnames for the web process (the default <app>.<domain> stays).
domains:
  - hello.example.test
```

## Fields

| Field | Meaning |
| --- | --- |
| `app` | Project name; used when `--app` is not given. Created with `shpyrd apps create <name>` (`--save` writes this file). |
| `processes.<type>.port` | Port the process listens on. `web` defaults to 8080 and is published through the URL; other types get no port unless set. `PORT` is injected. |
| `processes.<type>.replicas` | Pin the number of instances. Without it, `shpyrd scale` values are kept across deploys (default 1). |
| `processes.<type>.cpu`, `memory` | Process size (Kubernetes limits). Defaults 1 CPU and 512Mi; requests are 100m and 128Mi, capped by the limits. Metrics show usage as a percentage of these. |
| `processes.<type>.command`, `args` | Override the command. By default `web` runs the image entrypoint and other types run `/cnb/process/<type>`, the process the buildpacks recorded under that name. |
| `build.env` | Environment for the build (buildpack configuration such as `BP_GO_TARGETS`, `BP_JVM_VERSION`, `BP_NODE_RUN_SCRIPTS`). Runtime config vars are set with `shpyrd secrets`, not here. |
| `build.builder` | kpack `ClusterBuilder` to use. |
| `domains` | Additional hostnames; certificates are issued for each. |

## Process types and buildpacks

The buildpacks decide which process types an image has:

- **Go**: one per build target; use `BP_GO_TARGETS=./cmd/a:./cmd/b` to build several. The first is the default (`web`).
- **Node.js**: `web` from `npm start`/`package.json`; other types via a `Procfile`.
- **Java, Python, Ruby, .NET**: the framework's entry point becomes `web`; add a `Procfile` for others.
- **Procfile**: `worker: bundle exec sidekiq` adds a `worker` type on any stack.

`processes` in `shpyrd.yaml` tells shpyrd which of those types to run and how many instances; the names must match what the image provides.
