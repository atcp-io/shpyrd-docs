---
title: Roadmap
description: What the MVP does today and what comes next.
---

Shpyrd is pre-alpha. The MVP described in [RFC-0001](https://github.com/atcp-io/shpyrd/blob/main/rfcs/0001-mvp-local-platform.md) is complete on a local kind cluster; the items below are ordered by what users hit first. Priorities move with feedback in [GitHub issues](https://github.com/atcp-io/shpyrd/issues). {% .lead %}

## Done (MVP)

- Runlevel installer with an embedded local profile: cert-manager and a development CA, ingress-nginx, in-cluster registry, kpack with multi-arch Paketo buildpacks, Prometheus and Grafana, the shpyrd server.
- `App` controller: buildpack builds from archives or Git, sized processes, URLs with TLS, releases with config snapshots, config-restoring rollbacks, failure detection.
- CLI: cluster lifecycle, projects, deploy, config vars, scale, logs, releases, rollback.
- Dashboard: projects, activity, metrics per process, streamed logs, builds, write-only config vars, cluster capacity, token authentication, light/dark themes.

## Next

- **Published binaries and images.** Release builds of the CLI (Homebrew, curl installer) and `ghcr.io/atcp-io/shpyrd-server` so `make cli` is no longer required.
- **AWS profile.** EKS with the AWS Load Balancer Controller, ExternalDNS/Route53, ACM (or Let's Encrypt) and ECR, reusing the modules proven in the 2023 proofs of concept.
- **Dockerfile builds.** For repositories that already have one, next to buildpacks.
- **Log aggregation.** Loki and Alloy so logs survive restarts and can be searched over time.
- **Add-ons.** Managed Postgres (CloudNativePG) and Redis attachable to a project as config vars, Heroku-style.
- **Autoscaling and cost.** HPA/KEDA per process type; per-project cost from resource requests.
- **Users and access.** OIDC login (Okta, GitHub) and per-project permissions instead of the single admin token.
- **GitOps export as a first-class flow.** Keep `cluster export` in step with Flux and Argo CD conventions.
- **`git push shpyrd main`.** A Git receiver on top of the existing archive deploy path.
- **Agents.** First-class support for long-running, non-HTTP processes (queues, schedules, LLM agents) with the same deploy, config and observability story.

## Not planned for now

- Emulating cloud services locally (LocalStack and similar). Environment profiles abstract them instead.
- A hosted control plane. Shpyrd runs inside your cluster.
