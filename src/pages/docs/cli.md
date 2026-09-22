---
title: CLI reference
description: Every shpyrd command and its flags.
---

`shpyrd` uses your kubeconfig (`--kubeconfig`, `--context` on every command; `-v` for verbose output). Project commands take `--project <name>` or read `app:` from `shpyrd.yaml` in the current directory. {% .lead %}

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

## Projects

| Command | What it does |
| --- | --- |
| `shpyrd projects create <name>` | Create the project. `--domain` extra hostnames, `--save` writes `shpyrd.yaml`. (`shpyrd apps` still works as an alias.) |
| `shpyrd projects list` | Table of projects: phase, release, URL, age. |
| `shpyrd projects info <name>` | Phase and message, URL, build digest, source, processes (with sizes and failing reasons), recent releases. |
| `shpyrd projects destroy <name>` | Delete the project and its namespace (`--yes`). |

## Deploying and running

| Command | What it does |
| --- | --- |
| `shpyrd deploy` | Archive the committed tree of the current directory, upload, build and release. `--working-tree` deploys the directory as is; `--git <url> --ref <rev> --path <dir>` builds from Git; `--image <ref>` runs a prebuilt image; `--no-wait` returns immediately. Applies `shpyrd.yaml` (processes, sizes, build env, domains). |
| `shpyrd scale web=N worker=M` | Set instance counts per process type. |
| `shpyrd resize web=SIZE worker=SIZE` | Set instance sizes per process type (a release). |
| `shpyrd sizes list` | The cluster's instance size catalog with kind, cpu, burst and memory. |
| `shpyrd sizes set <name> --kind shared\|dedicated --cpu <cores> --memory <bytes> [--default]` | Add or change a size; processes using it are resized. |
| `shpyrd sizes delete <name>`, `shpyrd sizes default <name>` | Remove a size (not the default), choose the default. |
| `shpyrd secrets set K=V ...` | Set config vars (new release, rolling restart). |
| `shpyrd secrets unset K ...` | Remove config vars. |
| `shpyrd secrets list` | Names and last-updated times. Values are never printed. |
| `shpyrd logs` | Tail logs of every instance (`web.1`, `worker.2`...). `-f` follow, `-p <process>`, `-n <lines>`, `--build` for the latest build output. |
| `shpyrd releases` | Release history with digests and descriptions. |
| `shpyrd rollback [N]` | Re-release N (default: the previous release) with its build and config vars. Refused while another release is rolling out unless `--force`; `--no-wait`. |
| `shpyrd open` | Open the project URL in the browser. |

## Where things are

| | |
| --- | --- |
| `~/.shpyrd/ca/` | development root CA (`rootCA.pem`, key) |
| `~/.kube/config` | kind writes the `kind-shpyrd` context here |
| namespace `shpyrd-system` | server, registry, admin token, install record |
| namespace `app-<name>` | one per project: App, Deployments, Services, Ingress, kpack Image and Builds, config var Secret and release snapshots |
