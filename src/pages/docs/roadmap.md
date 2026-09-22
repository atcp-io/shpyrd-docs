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
- Instance sizes: a cluster-wide catalog of shared and dedicated sizes, `shpyrd resize`, sizes in releases.

## Done (phase B)

Designed in [RFC-0003](https://github.com/atcp-io/shpyrd/blob/main/rfcs/0003-projects-and-resources.md), [0004](https://github.com/atcp-io/shpyrd/blob/main/rfcs/0004-dockerfile-builds.md), [0005](https://github.com/atcp-io/shpyrd/blob/main/rfcs/0005-shell-and-one-off-commands.md) and [0006](https://github.com/atcp-io/shpyrd/blob/main/rfcs/0006-persistent-volumes.md):

- **Dockerfile builds** with rootless BuildKit Jobs next to buildpacks, auto-detected, with build args, targets and a layer cache.
- **Shell and one-off commands**: `shpyrd shell` into running instances, `shpyrd run` for migrations and scripts.
- **Persistent volumes**: single-instance (block) and shared volumes mounted from `shpyrd.yaml`, with the access-mode rules enforced.
- **Projects with several resources**: one resource list per project in the CLI, API and dashboard, and the binding plumbing that turns an attached resource into config vars.

## Done (phase C)

Designed in [RFC-0002](https://github.com/atcp-io/shpyrd/blob/main/rfcs/0002-extension-model.md) and [RFC-0007](https://github.com/atcp-io/shpyrd/blob/main/rfcs/0007-authentication.md):

- **Extensions**: optional capabilities compiled in and enabled per cluster (`shpyrd extensions enable`), each contributing an installer component, controllers, API routes and CLI commands.
- **Sign-in with accounts**: the server as an OpenID Connect relying party and the `auth-local` extension (Dex with local email/password accounts, `shpyrd users`, a Users page).

## Next

The remaining phases are designed in the [RFC index](https://github.com/atcp-io/shpyrd/blob/main/rfcs/README.md): teams and roles mirrored into Kubernetes RBAC with production hardening (RFC-0008), the fuller account system and company identity providers (RFC-0007 steps 3.2 and 3.3), and Postgres and Redis resources (RFC-0009, RFC-0010).

- **Teams, roles and security** (phase D). Project membership, roles mirrored into Kubernetes RBAC, network policies, quotas, audit log; Okta and email/password.
- **Postgres and Redis** (phase E). CloudNativePG and Valkey resources attached to apps as config vars (`shpyrd attach db`), plus shared storage for volumes.
- **Published binaries and images.** Release builds of the CLI (Homebrew, curl installer) and `ghcr.io/atcp-io/shpyrd-server` so `make cli` is no longer required.
- **AWS profile.** EKS with the AWS Load Balancer Controller, ExternalDNS/Route53, ACM (or Let's Encrypt) and ECR, reusing the modules proven in the 2023 proofs of concept.
- **Log aggregation.** Loki and Alloy so logs survive restarts and can be searched over time.
- **Autoscaling and cost.** HPA/KEDA per process type; per-project cost from resource requests.
- **Users and access.** Local users, email/password and OIDC (Okta, GitHub) through one relying-party implementation; teams and roles per project, mirrored into Kubernetes RBAC.
- **GitOps export as a first-class flow.** Keep `cluster export` in step with Flux and Argo CD conventions.
- **`git push shpyrd main`.** A Git receiver on top of the existing archive deploy path.
- **Agents.** First-class support for long-running, non-HTTP processes (queues, schedules, LLM agents) with the same deploy, config and observability story.

## Not planned for now

- Emulating cloud services locally (LocalStack and similar). Environment profiles abstract them instead.
- A hosted control plane. Shpyrd runs inside your cluster.
