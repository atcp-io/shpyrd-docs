---
title: Extensions and sign-in
description: Optional capabilities switched on per cluster, and how people sign in to the dashboard with their own accounts.
---

Shpyrd keeps its core small and adds optional capabilities as **extensions**: Go packages compiled into the binaries and switched on per cluster. The first one gives the dashboard real user accounts. {% .lead %}

## Extensions

```shell
shpyrd extensions list
shpyrd extensions enable auth-local
shpyrd extensions disable auth-local --yes
```

```
NAME        STATUS    COMPONENT  DESCRIPTION
auth-local  enabled   dex        Sign in with email and password: a bundled Dex issuer stores local accounts (shpyrd users add)
```

Enabling installs the extension's component with the same runlevel installer as the base stack (ordering, readiness waits, install record) and restarts the server with the extension; the choice is recorded in the cluster, so `shpyrd cluster init` and `shpyrd cluster status` keep it. Disabling removes the component and is refused while resources of the extension still exist. The **Cluster** page lists every extension with its state.

Extensions contribute an installer component, resource types with controllers, API routes, CLI commands and login providers through a few small Go interfaces ([RFC-0002](https://github.com/atcp-io/shpyrd/blob/main/rfcs/0002-extension-model.md)). Databases (Postgres, Redis) and shared storage arrive as extensions.

## Signing in with an account

Out of the box the dashboard is protected by the **admin token** (`shpyrd cluster token`), which is right for one developer and for automation (and can be switched off later, see [the admin token](/docs/access#the-admin-token)). With the `auth-local` extension, people get their own accounts:

```shell
shpyrd extensions enable auth-local
shpyrd users add ada@example.com --name "Ada Lovelace"   # prompts for the password
shpyrd users list
shpyrd users passwd ada@example.com
shpyrd users rm ada@example.com
```

The login page then offers **Sign in with email and password** next to the token. Signed-in users see who they are in the header and can sign out; a **Users** page lets platform admins add, reset and remove accounts (the same operations as the CLI). What each account may do is decided by [teams and roles](/docs/access).

### How it works

- The extension installs [Dex](https://dexidp.io), an OpenID Connect issuer, at `https://auth.<domain>` with a certificate from the cluster issuer. Accounts are Dex objects in the cluster (bcrypt hashes), so they survive restarts, upgrades and even disabling the extension.
- The shpyrd server is an OpenID Connect relying party: authorization code flow with PKCE, an HttpOnly session cookie, a CSRF token on every change, sessions that expire after 12 hours idle or 7 days. Because the relying party is generic, a company identity provider (Okta, Google, GitHub through Dex) is configuration, not code ([RFC-0007](https://github.com/atcp-io/shpyrd/blob/main/rfcs/0007-authentication.md)).
- The CLI keeps using your kubeconfig; cluster operations are not affected by dashboard accounts.

{% callout title="Local cluster note" %}
The kind cluster serves `auth.127.0.0.1.nip.io` with the development CA, so run `shpyrd cluster trust-ca` once or your browser will warn on the Dex login page. On other hosts, `shpyrd cluster init --domain` decides the hostname.
{% /callout %}
