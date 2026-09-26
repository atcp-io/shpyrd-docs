---
title: Teams, roles and security
description: Who can do what on a project, how it maps to Kubernetes RBAC, and the isolation and audit that come with every project.
---

Identity says who you are; roles say what you may do. Projects grant roles to users and teams, the API enforces them, the dashboard hides what a role cannot do, and a controller mirrors the grants into Kubernetes RBAC. {% .lead %}

## Roles

| Role | Sees | Does |
| --- | --- | --- |
| `user` | the app itself (through the edge), the launcher | opens the app; nothing in the builder dashboard — see [Sign-in for your app](/docs/app-access) |
| `viewer` | overview, releases, builds, logs, metrics, config var **names**; opens the app | nothing |
| `developer` | everything a viewer sees | deploy, roll back, scale, resize, set and unset config vars, shell and one-off commands |
| `admin` | + members, resources | attach and detach resources, create volumes, manage members, destroy the project |
| `platform-admin` | everything, cluster page, extensions, teams, users | everything |
| `platform-viewer` | everything, read-only | nothing |

The first four are granted **per project**, to a user (by email) or to a **team**; every operating role opens the app too. The platform roles are carried by teams. A refusal is a plain sentence: *your role on project shop is developer: it cannot destroy the project*.

## Teams and members

```shell
shpyrd teams create platform --platform-role platform-admin --member you@example.com
shpyrd teams create web --member ada@example.com --group engineering
shpyrd teams add web bob@example.com
shpyrd teams list

shpyrd members add shop --team web --role developer
shpyrd members add shop --user guest@example.com --role viewer
shpyrd members list shop
shpyrd members remove shop --user guest@example.com
```

Team `groups` are names from your identity provider's groups claim: a company directory group maps to a team without listing people twice. The built-in team **everyone** holds every person who has signed in — grant it the `user` role and the whole company can open an app (`shpyrd members add intranet --team everyone --role user`); it cannot be edited or deleted, and kubectl bindings skip it. The dashboard has the same operations: the **Workspace** page (Teams tab) for platform admins and a **Members** card on every project for its admins.

Teams and grants live in the platform's **control-plane database** (a small PostgreSQL the base stack runs as `control-plane-db`, or the managed database named by `SHPYRD_DATABASE_URL`), together with the workspace's record of who has signed in (the Workspace page, People tab). Installs made before v0.4 kept them as Kubernetes objects (`Team`, `ProjectMember`); the first server start after the upgrade copies them into the database and marks the objects migrated — nothing to do by hand. The platform backup carries the database's content (`shpyrd cluster backups`).

{% callout title="Before the first team" %}
A fresh cluster has no teams or members, and every signed-in user is a platform admin so nothing is locked. Creating the first team or member switches enforcement on; the CLI says so, and the dashboard shows a notice until then. Put yourself in a `platform-admin` team first. The admin token is always a platform admin.
{% /callout %}

## Suspending someone

The People tab of the Workspace page (or `PATCH /api/workspace/people/<email>` with `{"status":"suspended"}`) switches a person off at once: no role anywhere, no app opens, sign-in refused — until reactivated. Removing them from the identity provider does the same at the session's end; suspension is for right now.

## Kubernetes RBAC mirror

For every project namespace the controller keeps `RoleBinding`s (`shpyrd-viewer`, `shpyrd-developer`, `shpyrd-admin`) bound to the fixed ClusterRoles `shpyrd-project-*`, with the users (by email) and groups holding each role; platform roles become `ClusterRoleBinding`s. The Kubernetes roles grant the same verbs the dashboard allows (developers can update the App and open shells in its instances; config var Secrets stay write-only), never more than shpyrd itself has.

Configure your API server with the same OIDC issuer (`--oidc-issuer-url`, `--oidc-username-claim=email`, `--oidc-groups-claim=groups`) and `kubectl` users get exactly the dashboard's view. The local kind cluster is not configured this way out of the box; the bindings are still created and visible with `kubectl get rolebindings -n app-<project>`.

## Isolation and hardening

Every project namespace gets:

- **A network policy.** Ingress only from the project's own instances, the ingress controller (your URL) and the monitoring namespace (metrics). Egress to the project, to platform namespaces (DNS, the registry, bound services) and to the internet, never to other projects. Projects talk to each other only through what a binding exposes.
- **Pod security.** The `restricted` Pod Security Standard is applied in warn and audit mode, and shpyrd runs every process (and every `shpyrd run` instance) as a non-root user with all capabilities dropped and the runtime's default seccomp profile. Buildpack images comply already; a Dockerfile needs a numeric `USER` (`USER 1000`): a named user cannot be verified by Kubernetes and is refused with an explanation, as is an image running as root.
- **Hardened dashboard.** HttpOnly session cookies with CSRF tokens, security headers with a strict Content Security Policy, and a rate limit on sign-in.

## The admin token

The token created at install time is a shared credential with full platform-admin rights, meant for bootstrap and automation. Obtaining it requires reading Secrets in `shpyrd-system` (`shpyrd cluster token`), which is cluster-admin access; the risk is in the copies you hand out. Keep it in check:

- `shpyrd cluster dashboard` does **not** put the token in the browser: it mints a one-time login ticket (a hashed Secret valid for 60 seconds) that the browser redeems for a normal session, attributed to you as `user@host` in the audit trail.
- `shpyrd cluster token --rotate` replaces it and restarts the server; update automation that used the old value.
- `shpyrd cluster token --disable` switches it off once accounts exist and a `platform-admin` team has members: the API then refuses the token and everyone signs in with an account. `--enable` turns it back on.
- Wrong tokens are audited and throttled per client (20 attempts a minute, after which even the right token waits).

## Audit trail

Every mutation is recorded as `{who, what, target, detail, when, from, via}`: deploys, rollbacks, scaling and resizing, config var changes (names, never values), shells and one-off commands, volume and membership changes, project creation and destruction, from the dashboard and API (`via: api`, with the signed-in user) and from the CLI (`via: cli`, with the local user and host). The project page shows the recent actions; `GET /api/projects/{slug}/audit` returns them. Entries are Kubernetes Events, kept for the API server's event TTL (an hour by default) until durable storage arrives.

## Not yet

Resource quotas per project, image signing and CVE reporting, per-user API tokens and an enforcing Pod Security mode are on the roadmap ([RFC-0008](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0008-teams-roles-and-security.md)).
