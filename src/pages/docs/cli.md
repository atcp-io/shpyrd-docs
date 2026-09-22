---
title: CLI reference
description: Every shpyrd command and its flags.
---

`shpyrd` uses your kubeconfig (`--kubeconfig`, `--context` on every command; `-v` for verbose output). Project commands take `--project <name>` or read `project:` from `shpyrd.yaml` in the current directory. Commands contributed by extensions (`shpyrd users`) explain themselves when the extension is not enabled. {% .lead %}

## Cluster

| Command | What it does |
| --- | --- |
| `shpyrd cluster create` | Create a kind cluster and install the base stack. `--name`, `--workers`, `--image`, `--http-port`, `--https-port`, `--domain`, `--profile`, `--skip`, `--only`, `--set SHPYRD_X=y`, `--no-init`. |
| `shpyrd cluster init` | Install or upgrade the base stack on the current context. Same profile flags; `--yes` for non-kind contexts. Re-running is idempotent. |
| `shpyrd cluster status` | Health of every component, with versions and install times. Exit code 1 when something is not ready. |
| `shpyrd cluster dashboard` | Open the dashboard in the browser, signed in with the admin token. `--no-open` just prints URL and token. |
| `shpyrd cluster token` | Print the admin token (Secret `shpyrd-system/shpyrd-admin-token`). |
| `shpyrd cluster trust-ca` | Install the development root CA in the OS trust store (`--ca-dir`). |
| `shpyrd cluster export` | Render the base stack manifests to a directory for GitOps tooling (`-o`, profile flags). |
| `shpyrd cluster destroy` | Delete the kind cluster (`--name`, `--yes`). |
| `shpyrd extensions list` | Extensions known to this build and whether they are enabled on the cluster. |
| `shpyrd extensions enable <name>` | Install the extension's component and restart the server with it (`--set`). Also `cluster init --enable <name>`. |
| `shpyrd extensions disable <name>` | Remove the component (`--yes`); refused while resources of the extension exist. |
| `shpyrd users add <email>` | Create a local account (extension `auth-local`); `--name`, `--password` (prompted when omitted). |
| `shpyrd users list`, `passwd <email>`, `rm <email>` | Manage local accounts. |

## Projects

| Command | What it does |
| --- | --- |
| `shpyrd projects create <name>` | Create the project. `--domain` extra hostnames, `--save` writes `shpyrd.yaml`. (`shpyrd apps` still works as an alias.) |
| `shpyrd projects list` | Table of projects: phase, release, URL, age. |
| `shpyrd projects info <name>` | Phase and message, URL, build digest, source, processes (with sizes and failing reasons), recent releases, and every resource of the project (app, attached resources, volumes). |
| `shpyrd projects destroy <name>` | Delete the project and its namespace (`--yes`). |

## Deploying and running

| Command | What it does |
| --- | --- |
| `shpyrd deploy` | Archive the committed tree of the current directory, upload, build and release. `--working-tree` deploys the directory as is; `--git <url> --ref <rev> --path <dir>` builds from Git; `--dockerfile [path]` builds the Dockerfile (auto-detected for local deploys); `--image <ref>` runs a prebuilt image; `--no-wait` returns immediately. Applies `shpyrd.yaml` (processes, sizes, build, domains). |
| `shpyrd scale web=N worker=M` | Set instance counts per process type. |
| `shpyrd resize web=SIZE worker=SIZE` | Set instance sizes per process type (a release). |
| `shpyrd sizes list` | The cluster's instance size catalog with kind, cpu, burst and memory. |
| `shpyrd sizes set <name> --kind shared\|dedicated --cpu <cores> --memory <bytes> [--default]` | Add or change a size; processes using it are resized. |
| `shpyrd sizes delete <name>`, `shpyrd sizes default <name>` | Remove a size (not the default), choose the default. |
| `shpyrd secrets set K=V ...` | Set config vars (new release, rolling restart). |
| `shpyrd secrets unset K ...` | Remove config vars. |
| `shpyrd secrets list` | Names and last-updated times, plus variables provided by attached resources. Values are never printed. |
| `shpyrd shell [-- cmd...]` | Interactive shell in a running instance (`--process`, `--instance web.2`); with a command, runs it and returns its exit code. |
| `shpyrd run <cmd...>` | One-off instance of the current release with the config vars: streams output, returns the exit code, removes the instance. `--size`, `--detach`. |
| `shpyrd volumes create <name> --size 5Gi` | Create a persistent volume in the project (`--class`, `--shared`). |
| `shpyrd volumes list` | Volumes with size, mode, status and what mounts them. |
| `shpyrd volumes resize <name> --size 10Gi` | Grow a volume (when the storage class allows expansion). |
| `shpyrd volumes delete <name>` | Delete a volume and its data (`--yes`; `--force` while mounted). |
| `shpyrd logs` | Tail logs of every instance (`web.1`, `worker.2`...). `-f` follow, `-p <process>`, `-n <lines>`, `--build` for the latest build output. |
| `shpyrd releases` | Release history with digests and descriptions. |
| `shpyrd rollback [N]` | Re-release N (default: the previous release) with its build and config vars. Refused while another release is rolling out unless `--force`; `--no-wait`. |
| `shpyrd open` | Open the project URL in the browser. |

## Where things are

| | |
| --- | --- |
| `~/.shpyrd/ca/` | development root CA (`rootCA.pem`, key) |
| `~/.kube/config` | kind writes the `kind-shpyrd` context here |
| namespace `shpyrd-system` | server, registry, admin token, install record, sessions mirror, Dex and its accounts when `auth-local` is enabled |
| namespace `app-<name>` | one per project (label `shpyrd.io/project`): App, Volumes and their claims, Deployments, Services, Ingress, kpack Image and Builds or BuildKit Jobs, config var Secret, `<app>-bindings` and release snapshots |
