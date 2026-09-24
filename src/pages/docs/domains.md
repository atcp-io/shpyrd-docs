---
title: Domains and exposure
description: The hostname every project gets, custom domains you own, and which load balancer serves a project.
---

Every project is served at `<slug>.<cluster domain>` with a trusted certificate. Two settings change how the world reaches it: **custom domains** add hostnames you own, and **exposure** decides whether the public or the private load balancer serves the project. {% .lead %}

## The project hostname

`shop.oci.example.com` on a cloud profile, `shop.shpyrd.test` or `shop.127.0.0.1.nip.io` locally: the slug plus the cluster domain. It is always served, it is what `shpyrd open` opens, and it is the target a custom domain's DNS record points at. On cloud profiles with a DNS provider, one wildcard certificate covers every project hostname, so a new project is trusted from its first request; without one, each project gets its own certificate from Let's Encrypt (cloud) or the platform CA (local).

## Custom domains

```shell
shpyrd domains add www.myprod.com --project shop
```

```
Added www.myprod.com to shop. Create one DNS record at your provider:

  CNAME    www.myprod.com                           -> shop.oci.example.com
  or, for a zone apex that cannot hold a CNAME:
  A        www.myprod.com                           -> 147.15.59.84

Waiting for www.myprod.com to resolve here and for its certificate ... done.

https://www.myprod.com serves shop.
```

That is the whole flow, the same as on Heroku or Render: pointing DNS at the platform is the proof that you control the domain. No verification record, no button to click. Once the name resolves to the front door, cert-manager proves control to Let's Encrypt over HTTP and issues the certificate; the command waits (Ctrl-C keeps the domain; the certificate still comes) and the Domains card on the project page shows the state of each host, refreshing on its own.

| Host | Record to create |
| --- | --- |
| a subdomain (`www.myprod.com`, `app.acme.io`) | `CNAME` to the project hostname |
| a zone apex (`myprod.com`) | `A` to the front door's address (the reserved address on Oracle Cloud), or `ALIAS`/`ANAME` to the project hostname where the provider offers it (DNSimple, Cloudflare, Route 53 alias records) |

Each custom domain has its own certificate, so a domain whose DNS is not ready yet never affects the project hostname or the other domains. The card and `shpyrd domains list` report, per host:

```
DOMAIN            DNS      CERTIFICATE  NOTE
www.myprod.com    ok       ready        serving
myprod.com        missing  issuing      create: CNAME myprod.com -> shop.oci.example.com (or A -> 147.15.59.84)
```

`dns` is `ok` (resolves to this cluster's front door), `missing` (no record yet), `wrong` (points elsewhere; the note says what to change) or `unknown` (the lookup failed); `certificate` is `issuing`, `ready`, `failed` (with the certificate authority's reason) or `wildcard` for a host the platform's wildcard already covers.

Rules: a hostname is served by one project only - adding one another project has names that project; wildcards (`*.myprod.com`) are not accepted, since Let's Encrypt issues those only through DNS in your zone; the platform's own hostnames are refused. `shpyrd domains rm www.myprod.com` stops serving a domain and removes its certificate. `domains:` in [`shpyrd.yaml`](/docs/shpyrd-yaml) declares the same list for a repository.

Let's Encrypt allows 50 certificates per registered domain per week and five failed validations per hostname per hour; cert-manager retries on its own, so a domain whose record arrives an hour late still comes up.

## Exposure: external or internal

Cloud profiles can run two front doors: the **external** load balancer with a public address, and an **internal** one, reachable only from inside the network (VPN, peered VCN, bastion). Every project and the platform itself are external by default; a project moves with one setting:

```shell
shpyrd exposure internal --project backoffice
shpyrd exposure external --project backoffice
```

or the Exposure badge on the project page, or `exposure: internal` in `shpyrd.yaml`. Changing it re-renders the project's Ingress on the other controller - no release, no downtime for the instances - and the DNS record for its hostname follows on the provider's next sync, pointing at the internal address. Certificates keep working: the wildcard covers the hostname, and a custom domain's certificate is issued through the front door the domain points at.

The internal controller exists when the cluster was installed with `--internal-lb-subnet <subnet ocid>` (Oracle Cloud: the private load balancer subnet that `contrib/oci/terraform` creates). Without it, setting a project internal is refused with an explanation. The dashboard, sign-in page and Grafana can move behind the internal front door too, once a team has the network to reach them: `shpyrd cluster init --platform-exposure internal`.

The cluster page shows both front doors with their addresses, and the projects list marks internal projects.

## Local profile

The local profile has one front door (kind's host ports or your Caddy) and no private network, so `exposure` is accepted but both values map to the same controller. Custom domains work the same way as on cloud profiles: point a name at the machine and the platform CA issues the certificate (`shpyrd cluster trust-ca` once).
