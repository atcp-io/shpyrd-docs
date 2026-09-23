---
title: Roadmap
description: What is done, what is decided and what is still a proposal, with a link to the RFC behind every line.
---

Every line of the roadmap is an RFC in the [shpyrd repository](https://github.com/shpyrd-io/shpyrd/tree/main/rfcs): **done** is merged, **ready to implement** is decided and waiting for someone to pick it up, **proposal** still has open questions (each with a default). Priorities move with feedback in [GitHub issues](https://github.com/shpyrd-io/shpyrd/issues). {% .lead %}

## Platform

| Item | RFC | Status |
| --- | --- | --- |
| Local platform: installer, App controller, CLI, dashboard | [RFC-0001](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0001-mvp-local-platform.md) | done |
| Extensions enabled per cluster | [RFC-0002](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0002-extension-model.md) | done |
| Published binaries, images and CI (Homebrew, curl installer, ghcr image) | [RFC-0045](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0045-published-binaries-and-ci.md) | done |
| Object storage extension (MinIO locally, S3 on cloud) | [RFC-0046](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0046-object-storage.md) | ready to implement |
| Platform backup and restore (encrypted, to object storage) | [RFC-0037](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0037-platform-backup-and-restore.md) | proposal |
| Local names and front door: `*.shpyrd.test` via dnsmasq, an existing Caddy on 443 as the front door | [RFC-0057](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0057-local-names-and-front-door.md) | ready to implement |
| Project identity: display names, `/projects/<slug>` URLs, no "pod" wording | [RFC-0011](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0011-project-identity-and-product-language.md) | done |
| Global config vars for every project | [RFC-0016](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0016-global-config-vars.md) | proposal |
| Project quotas | [RFC-0042](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0042-project-quotas.md) | ready to implement |
| Cost visibility | [RFC-0048](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0048-cost-visibility.md) | ready to implement |
| Workspaces (grouping of projects) | [RFC-0033](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0033-workspaces.md) | proposal |

## Deploying

| Item | RFC | Status |
| --- | --- | --- |
| Projects and resources, bindings, attach/detach | [RFC-0003](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0003-projects-and-resources.md) | done |
| Dockerfile builds with BuildKit | [RFC-0004](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0004-dockerfile-builds.md) | done |
| Shell and one-off commands | [RFC-0005](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0005-shell-and-one-off-commands.md) | done |
| Persistent volumes (single-instance) | [RFC-0006](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0006-persistent-volumes.md) | done |
| Shared volumes (`storage-rwx`) | [RFC-0041](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0041-shared-volumes.md) | ready to implement |
| Private repositories (tokens, deploy keys) | [RFC-0017](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0017-git-credentials.md) | proposal |
| Auto-deploy on push (webhooks, polling) | [RFC-0018](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0018-repository-monitoring.md) | proposal |
| GitHub App: connect once, pick repositories, statuses | [RFC-0054](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0054-github-app.md) | ready to implement |
| Health checks and zero-downtime rollouts | [RFC-0019](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0019-health-checks-and-rollouts.md) | proposal |
| Autoscaling mode (min/max, HPA, KEDA) | [RFC-0047](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0047-autoscaling.md) | ready to implement |
| Maintenance mode | [RFC-0020](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0020-maintenance-mode.md) | proposal |
| Run history and scheduled tasks | [RFC-0024](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0024-runs-and-scheduled-tasks.md) | proposal |
| Web terminal | [RFC-0026](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0026-web-terminal.md) | ready to implement |
| Builds namespace and enforce-mode Pod Security | [RFC-0043](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0043-builds-namespace-and-pod-security.md) | ready to implement |

## Data stores

| Item | RFC | Status |
| --- | --- | --- |
| Postgres (CloudNativePG) | [RFC-0009](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0009-postgres-resource.md) | done |
| Redis and Valkey | [RFC-0010](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0010-redis-resource.md) | done |
| Postgres backups and point-in-time recovery | [RFC-0038](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0038-postgres-backups-and-pitr.md) | ready to implement |
| Postgres pooling, credential rotation, resize | [RFC-0039](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0039-postgres-pooling-rotation-resize.md) | ready to implement |
| Redis high availability and metrics exporter | [RFC-0040](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0040-redis-ha-and-exporter.md) | proposal |
| Resource detail pages with their own metrics | [RFC-0028](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0028-resource-pages-and-metrics.md) | proposal |

## Observability

| Item | RFC | Status |
| --- | --- | --- |
| Structured (JSON) logs in the viewer and CLI | [RFC-0021](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0021-structured-logs.md) | ready to implement |
| Log pipeline: logs that survive restarts, history | [RFC-0022](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0022-log-pipeline.md) | proposal |
| Log drains (syslog, HTTPS) to any provider | [RFC-0023](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0023-log-drains.md) | proposal |
| Application metrics v2 (per instance, aggregation, totals) | [RFC-0027](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0027-application-metrics-v2.md) | ready to implement |
| OpenTelemetry collector and export | [RFC-0029](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0029-opentelemetry.md) | proposal |
| Tracing backend (Jaeger) and a Traces tab | [RFC-0056](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0056-tracing-backend.md) | ready to implement |
| Notifications: webhook, Slack, email | [RFC-0030](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0030-notifications.md) | proposal |
| Audit trail v2 (durable, cluster-wide, export) | [RFC-0025](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0025-audit-trail-v2.md) | proposal |

## Access

| Item | RFC | Status |
| --- | --- | --- |
| Sign-in with accounts (Dex, local users) | [RFC-0007](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0007-authentication.md) | done |
| Teams, roles, RBAC mirror, isolation, audit | [RFC-0008](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0008-teams-roles-and-security.md) | done |
| Sign-in experience: shpyrd's own sign-in page, local sign-in, sign-out at the issuer | [RFC-0012](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0012-sign-in-experience.md) | done |
| External identity providers: Okta and any OIDC issuer, GitHub and Google | [RFC-0058](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0058-external-identity-providers.md) | done |
| Email delivery | [RFC-0013](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0013-email-delivery.md) | proposal |
| Account lifecycle: invitations, reset, verification, lockout | [RFC-0014](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0014-account-lifecycle.md) | proposal |
| MFA and passkeys | [RFC-0053](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0053-mfa-and-passkeys.md) | ready to implement |
| Grafana behind shpyrd sign-in | [RFC-0015](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0015-grafana-sign-in.md) | proposal |
| Per-user API tokens | [RFC-0031](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0031-api-tokens.md) | ready to implement |
| API-first CLI and `shpyrd login` | [RFC-0052](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0052-api-first-cli-and-login.md) | ready to implement |
| MCP connector for AI agents | [RFC-0032](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0032-mcp-connector.md) | proposal |
| Supply chain and encryption at rest | [RFC-0044](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0044-supply-chain.md) | ready to implement |

## Cloud

| Item | RFC | Status |
| --- | --- | --- |
| Domains and certificates (wildcards, custom domains, Let's Encrypt) | [RFC-0034](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0034-domains-and-certificates.md) | proposal |
| AWS profile (EKS) | [RFC-0035](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0035-aws-profile.md) | proposal |
| DNS providers (Route53, Cloudflare) and internal/external load balancers | [RFC-0036](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0036-dns-providers-and-exposure.md) | proposal |

## Decided against, for now

| Item | RFC | Why |
| --- | --- | --- |
| GitOps export as a first-class flow | [RFC-0049](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0049-gitops-export.md) | the platform installs and upgrades itself; `shpyrd cluster export` stays as a plain rendering |
| `git push shpyrd main` | [RFC-0050](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0050-git-push-receiver.md) | `shpyrd deploy` and auto-deploy from the repository cover the workflows in use |
| Agents as a separate kind | [RFC-0051](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0051-agents-and-background-processes.md) | an agent is an app: a worker process, a run or a scheduled task, with everything apps get |
| Environments and promotion | [RFC-0055](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0055-environments-and-promotion.md) | deferred: environments are Git branches deploying to their own projects, pull requests promote |

## Not planned

- Emulating cloud services locally (LocalStack and similar). Environment profiles abstract them instead.
- A hosted control plane. Shpyrd runs inside your cluster.
