---
title: Installation
description: Create a local cluster with the shpyrd base stack, or install it on an existing Kubernetes cluster.
---

Shpyrd ships as a single CLI, `shpyrd`, that installs the platform on a Kubernetes cluster: a local kind cluster it creates for you, or a cluster you already have. {% .lead %}

## Requirements

- **Docker** (Docker Desktop on macOS/Windows, Docker Engine on Linux). Give it 6-8 GB of memory: the base stack idles around 3 GB and buildpack builds need headroom.
- **Go 1.27** to build the CLI, until binaries are published.
- Internet access for the first install (kind node image, Helm charts, buildpacks, ~2 GB) and for name resolution of the default `127.0.0.1.nip.io` domain.

{% callout type="warning" title="Docker Desktop and cgroup v1" %}
Recent Kubernetes releases refuse to run on cgroup v1. If Docker Desktop has the deprecated cgroup v1 setting enabled (`DeprecatedCgroupv1` in its settings), `shpyrd cluster create` detects it, applies a kubelet override and warns; switching Docker Desktop to cgroup v2 is recommended.
{% /callout %}

## Build the CLI

```shell
git clone https://github.com/shpyrd-io/shpyrd && cd shpyrd
make cli            # -> ./bin/shpyrd
./bin/shpyrd --help
```

Add `./bin` to your `PATH` or copy the binary somewhere on it.

## Create a local cluster

```shell
shpyrd cluster create
```

This runs [kind](https://kind.sigs.k8s.io) as a library to create a two-node cluster named `shpyrd` (one control-plane, one worker), then installs the base stack in dependency-ordered **runlevels**, waiting for each to be healthy:

| Level | Components |
| --- | --- |
| rc0 | Prometheus Operator CRDs |
| rc1 | cert-manager |
| rc2 | development CA `ClusterIssuer`, trust-manager, ingress-nginx (host ports 80/443), in-cluster registry |
| rc3 | kpack with the Paketo buildpacks builder, kube-prometheus-stack + Grafana |
| rc4 | shpyrd server (API, App controller, dashboard) |

The first run takes 10-20 minutes, mostly downloads. Re-running `cluster create` or `cluster init` on an existing cluster is idempotent and takes about 30 seconds.

Options worth knowing:

```shell
shpyrd cluster create --http-port 8080 --https-port 8443   # host ports 80/443 already in use
shpyrd cluster create --domain myapps.example.test          # wildcard domain resolving to your machine
shpyrd cluster create --workers 2                          # more kind worker nodes
shpyrd cluster create --skip monitoring                    # lighter install, no Prometheus/Grafana
shpyrd cluster create --no-init                            # only the kind cluster
```

## Trust the development CA

Certificates for `https://<project>.<domain>` are issued by a root CA generated on your machine (`~/.shpyrd/ca/rootCA.pem`) and stored in the cluster. Install it in your operating system trust store once:

```shell
shpyrd cluster trust-ca      # asks for sudo (macOS keychain / Linux ca-certificates)
```

Firefox keeps its own store: enable `security.enterprise_roots.enabled` in `about:config` or import the certificate.

## Check the installation

```shell
shpyrd cluster status
```

```
Profile: local  Version: dev  Domain: 127.0.0.1.nip.io  Updated: 2026-09-21T22:23:28Z

RUNLEVEL  COMPONENT        STATUS  VERSION  APPLIED
rc0       monitoring-crds  ready   32.0.0   ...
rc1       cert-manager     ready   v1.21.2  ...
rc2       ca-issuers       ready            ...
rc2       trust-manager    ready   v0.25.0  ...
rc2       ingress-nginx    ready   4.15.1   ...
rc2       registry         ready            ...
rc3       kpack            ready            ...
rc3       monitoring       ready   91.4.1   ...
rc4       shpyrd           ready            ...
```

Endpoints on the default domain:

- `https://shpyrd.127.0.0.1.nip.io` — dashboard (`shpyrd cluster dashboard` opens it signed in; `shpyrd cluster token` prints the admin token)
- `https://grafana.127.0.0.1.nip.io` — Grafana (`admin` / `shpyrd` on the local profile)
- `localhost:30050` — the in-cluster registry, for pushing images from your machine

## Install on an existing cluster

The installer works against any kubeconfig context:

```shell
shpyrd cluster init --context my-cluster --profile local --domain apps.example.test --yes
```

`--yes` is required for contexts that do not look like kind clusters. Only the `local` profile exists today; it assumes ingress-nginx can bind host ports on a node labelled `ingress-ready=true` and that the service subnet is `10.96.0.0/16` (the registry uses the fixed ClusterIP `10.96.0.50`). Cloud profiles (AWS first) are on the [roadmap](/docs/roadmap).

## Environment profiles

A **profile** describes the environment the base stack is built for and therefore how load balancing, DNS, TLS and the registry are provided:

| | `local` (today) | `aws` (planned) |
| --- | --- | --- |
| Load balancer | kind host ports 80/443 | AWS Load Balancer Controller |
| DNS | `*.127.0.0.1.nip.io` wildcard | Route53 via ExternalDNS |
| TLS | development CA issued by cert-manager | ACM Private CA / Let's Encrypt |
| Registry | in-cluster `registry:3` | ECR |

The dashboard's cluster page shows the installed profile.

## Export the manifests

For GitOps tooling (Flux, Argo CD) the same embedded manifests can be rendered to disk instead of applied:

```shell
shpyrd cluster export -o ./gitops --domain apps.example.test
```

## Remove everything

```shell
shpyrd cluster destroy     # deletes the kind cluster
```

Projects, images and configuration live inside the cluster and disappear with it; the development CA under `~/.shpyrd/ca` is kept so the next cluster is trusted immediately.
