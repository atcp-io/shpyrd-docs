---
title: Dashboard
description: The web UI - projects, releases, builds, logs, metrics and cluster capacity.
---

The dashboard is served by the shpyrd server inside the cluster at `https://shpyrd.<domain>`. Everything the CLI does for projects can be done there, and it is the place to watch what is happening. {% .lead %}

## Signing in

```shell
shpyrd cluster dashboard        # opens the browser signed in
shpyrd cluster token            # prints the admin token for the login screen
```

`cluster dashboard` puts the token in the URL fragment (`#token=...`), which browsers never send to servers; the UI stores it locally and removes it from the address bar. The token is generated once at install time and lives in Secret `shpyrd-system/shpyrd-admin-token`.

Themes: light, dark or system, from the selector in the header.

## Projects

The **Projects** page lists every project with its phase, current release, per-process health (`web 3/3`) and URL, plus counters for the cluster. **New project** creates one, optionally pointing it at a public Git repository to build and deploy immediately.

A project page has:

- **Header**: phase, process chips (green when all instances are on the current release and ready, amber while rolling out, red with a count when instances are failing), the URL, and the **Open**, **Deploy** (from Git) and **Destroy** actions.
- **Activity panel**, only when something is happening: live build output while building; per-process rollout progress ("1/3 on new release · 3 serving") while deploying; the container's reason and a one-click rollback when a release is not healthy.
- **Overview**: source, current release and build, per-process instances and size with scale buttons, and the **Releases** table (kind badge, what changed, build number, rollback button).
- **Metrics**: see below.
- **Logs**: every instance streamed live, named `web.1`, `worker.2`; filter box, pause/live, error and warning highlighting.
- **Builds**: build history with status, reason, source and duration; select one to read its full output (live while building), and which releases use it.
- **Config**: config var names and last-updated times; add, replace (blind), remove, or paste a `.env`. Values are never shown.

Actions that would start another release (Deploy, Rollback) are disabled while one is building or rolling out.

## Metrics

Modelled on what Heroku, Fly and Render show for an application:

| Chart | What it shows |
| --- | --- |
| Throughput | requests per second at the ingress, stacked by response class (2xx, 3xx, 4xx, 5xx) |
| Response time | p50, p95 and p99 latency at the ingress |
| Instances | running instances per process type (step chart) |
| CPU | usage as a percentage of each process' allocation, averaged over its instances; 100% means every instance saturating its CPU |
| Memory | working set as a percentage of each process' allocation |
| Network | pod traffic in and out |

Orange dashed lines mark releases; a red line marks 100%. Ranges: last hour, 6 hours, 24 hours, 7 days. Sizes (the 100% marks) are the `cpu`/`memory` of each process in [`shpyrd.yaml`](/docs/shpyrd-yaml), 1 CPU and 512 MiB by default.

Grafana, linked from the header, has the same data with the pre-provisioned "shpyrd / Web apps" dashboard and everything kube-prometheus-stack ships.

## Cluster

The **Cluster** page shows the environment profile, version and domain; **capacity**: CPU and memory **used** (what the machines are doing) versus **reserved** (what running processes have requested, which is what limits scheduling), in total and per node, with utilisation over time; the installed components with versions; and the Helm releases in the cluster.

## Security notes

- Every `/api` route requires the admin token except the health check, the public configuration and content-addressed source archives fetched by build pods.
- Config var values are write-only through the API and the UI.
- Image references and internal addresses (registry, blob storage) are not exposed; builds and releases are identified by digest.
- On the local profile the dashboard is only reachable from your machine.
