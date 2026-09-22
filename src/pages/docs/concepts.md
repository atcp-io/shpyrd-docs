---
title: Concepts
description: Projects, processes, builds, releases and config vars - the vocabulary shpyrd shares with Heroku and Fly.
---

Shpyrd borrows the vocabulary of Heroku and Fly and maps it onto Kubernetes objects you can always inspect with `kubectl`. {% .lead %}

## Project (App)

A project is one deployable codebase with a name, a source, process types and config vars. It is stored as an `App` custom resource (`shpyrd.io/v1alpha1`) in its own namespace, `app-<name>`, together with everything the controller creates for it. `kubectl get apps -A` lists them all.

```yaml
apiVersion: shpyrd.io/v1alpha1
kind: App
metadata:
  name: hello-world
  namespace: app-hello-world
spec:
  source:
    git: { url: https://github.com/atcp-io/shpyrd, revision: main }
    subPath: examples/hello
  processes:
    web: { port: 8080, replicas: 3 }
    worker: { replicas: 2 }
  build:
    env: [{ name: BP_GO_TARGETS, value: ./cmd/web:./cmd/worker }]
status:
  phase: Running
  url: https://hello-world.127.0.0.1.nip.io
  releases: [...]
```

Projects without a `web` process (workers, agents, schedulers) work the same way; they just get no URL.

## Processes

A **process type** is a way of running the build: `web` serves HTTP and receives `PORT`; anything else (`worker`, `scheduler`, `agent`) runs the command of the same name that the buildpacks recorded in the image. Each process type becomes a Deployment with its own **instance** count (`shpyrd scale web=3 worker=1`) and **size**: by default 1 CPU and 512 MiB (requests 100m / 128 MiB); override per process with `cpu` and `memory` in [`shpyrd.yaml`](/docs/shpyrd-yaml).

Instances are named the way Heroku names dynos: `web.1`, `web.2`, `worker.1`, in creation order. Logs and the dashboard use these names.

## Builds

A **build** compiles source into an OCI image with [Cloud Native Buildpacks](https://buildpacks.io) (Paketo), run inside the cluster by [kpack](https://github.com/buildpacks-community/kpack). A build happens for every new source: an uploaded archive, or a new commit on a Git branch (kpack polls). Builds are numbered (`#1`, `#2`, ...) and identified by the image digest; the dashboard shows every step (prepare, analyze, detect, restore, build, export) live.

Buildpacks detect the language from the repository (`go.mod`, `package.json`, `pom.xml`, `requirements.txt`, `Gemfile`, `*.csproj`, or static files) and produce one process type per entry point. A `Procfile` or buildpack-specific settings (`BP_*` variables in `build.env`) refine that, for example `BP_GO_TARGETS` to build several Go commands.

## Releases

A **release** is a build plus the config vars in effect, numbered `v1`, `v2`, ... Anything that changes what runs creates one:

| Kind | Example description | Creates a build? |
| --- | --- | --- |
| deploy | `Deploy 654f4925638e` | yes |
| config | `Set GREETING config var` | no, reuses the current build |
| rollback | `Rollback to v7` | no, reuses v7's build |

Each release records its build, its config snapshot (a Secret `<app>-release-vN`) and its process types. **Rollback** re-releases an earlier release exactly: its build is pinned and its config vars are restored. Rolling back to a release whose build predates a process type (say, before `worker` existed) cannot start that process; the dashboard warns before and the failure is reported plainly after.

The next `shpyrd deploy` unpins the build and continues from the new source.

## Config vars

Config vars are environment variables for every process, stored in Secret `<app>-env`. They are **write-only** in the product: `shpyrd secrets set/unset/list` and the dashboard show names and when each was last changed, never values. Changing them creates a `config` release and rolls the processes.

`PORT` is injected for processes with a port; plain, non-secret variables can also be declared in the App spec (`env`).

## Domains and TLS

A project's `web` process is published at `https://<name>.<cluster domain>` (additional `domains` can be declared). Certificates come from cert-manager: the development CA on the local profile, a public or private CA on cloud profiles.

## Environment profile

The **profile** chosen at install time (`local` today) says how load balancing, DNS, TLS and the registry are provided; see [Installation](/docs/installation#environment-profiles).

## Phases

A project's phase summarises its state:

| Phase | Meaning |
| --- | --- |
| Pending | no source or image yet |
| Building | a build is running (a previous release may still be serving) |
| Deploying | new instances are rolling out; previous ones keep serving until they are ready |
| Running | every process has its desired instances ready |
| Failed | the build failed, or new instances cannot start (the reason is shown, previous instances keep serving) |
