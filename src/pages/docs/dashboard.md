---
title: Dashboard
description: The web UI - projects, releases, builds, logs, metrics and cluster capacity.
---

The dashboard is served by the shpyrd server inside the cluster at `https://shpyrd.<domain>`. Everything the CLI does for projects can be done there, and it is the place to watch what is happening. {% .lead %}

## Signing in

`shpyrd cluster dashboard` opens the dashboard signed in with the admin token; the login page also accepts the token pasted by hand (`shpyrd cluster token`). With the `auth-local` extension enabled, the login page offers **Sign in with email and password** for accounts created with `shpyrd users add` or on the **Users** page; the header shows who is signed in and has a sign-out entry. See [Extensions and sign-in](/docs/extensions).

## Projects

The **Projects** page lists every project with its phase, current release, per-process health (`web 3/3`) and URL, plus counters for the cluster. **New project** creates one, optionally pointing it at a public Git repository to build and deploy immediately.

A project page has:

- **Header**: phase, process chips (green when all instances are on the current release and ready, amber while rolling out, red with a count when instances are failing), the URL, and the **Open**, **Deploy** (from Git, with buildpacks or a Dockerfile) and **Destroy** actions (the dialog lists every resource that goes, data-holding ones first).
- **Activity panel**, only when something is happening: live build output while building; per-process rollout progress ("1/3 on new release · 3 serving") while deploying; the container's reason and a one-click rollback when a release is not healthy.
- **Overview**: source and build strategy, current release and build, per-process instances with scale buttons and an instance size selector (pinned to one for processes mounting a single-instance volume), the **Resources** card (the app, its volumes and later databases, with status and what uses them; create, resize and delete volumes there) and the **Releases** table (kind badge, what changed, build number, rollback button).
- **Metrics**: see below.
- **Logs**: every instance streamed live, named `web.1`, `worker.2`; filter box, pause/live, error and warning highlighting.
- **Builds**: build history with status, strategy, reason, source and duration; select one to read its full output (live while building), and which releases use it.
- **Config**: config var names and last-updated times; add, replace (blind), remove, or paste a `.env`. Variables provided by attached resources are listed read-only with their provider. Values are never shown.

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

Orange dashed lines mark releases; a red line marks 100%. Ranges: last hour, 6 hours, 24 hours, 7 days. 100% is the process' instance size (its allocation); shared sizes can read above 100% while bursting.

Grafana, linked from the header, has the same data with the pre-provisioned "shpyrd / Web apps" dashboard and everything kube-prometheus-stack ships.

## Cluster

The **Cluster** page shows the environment profile, version and domain; **capacity**: CPU and memory **used** (what the machines are doing) versus **reserved** (what running processes have requested, which is what limits scheduling), in total and per node, with utilisation over time; the **instance size catalog** (add, change, delete sizes and pick the default); the **extensions** with their state; the installed components with versions; and the Helm releases in the cluster.

## Security notes

- Every `/api` route requires the admin token except the health check, the public configuration and content-addressed source archives fetched by build pods.
- Config var values are write-only through the API and the UI.
- Image references and internal addresses (registry, blob storage) are not exposed; builds and releases are identified by digest.
- On the local profile the dashboard is only reachable from your machine.
