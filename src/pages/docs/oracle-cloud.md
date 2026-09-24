---
title: Oracle Cloud (OKE)
description: Run shpyrd on Oracle Kubernetes Engine - the network and cluster from Terraform, then one command for the platform.
---

The `oci` profile installs shpyrd on Oracle Kubernetes Engine (OKE) with a public load balancer, Let's Encrypt certificates, an in-cluster registry, network policy enforcement and, optionally, automatic DNS. The reference infrastructure lives in [`contrib/oci`](https://github.com/shpyrd-io/shpyrd/tree/main/contrib/oci) as Terraform; the platform itself is `shpyrd cluster init`. {% .lead %}

Oracle Cloud went first among the cloud profiles for cost - the free tier and cheap flexible shapes - and because it exercises the harder path: a private API endpoint, private workers, CRI-O nodes. The layout is what the proof of concept at `oci.shpyrd.io` runs on ([RFC-0035](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0035-cloud-profiles.md)).

## What you get

| | |
| --- | --- |
| Network | a VCN (`10.0.0.0/16`) with private subnets for the Kubernetes API endpoint, the workers and the pods (VCN-native pod networking), a public and a private load balancer subnet, a Bastion subnet; internet, NAT and service gateways; network security groups with the rules OKE needs |
| Cluster | OKE, Basic (free control plane) or Enhanced, private API endpoint reached through the OCI Bastion service, one node pool of flexible-shape workers |
| Front doors | a public OCI flexible load balancer on a **reserved address** (survives cluster rebuilds); a private one for projects marked internal ([Domains and exposure](/docs/domains)) |
| Certificates | Let's Encrypt; with a DNS provider, one wildcard certificate for every project hostname |
| Registry | the in-cluster registry with TLS from the platform CA (no OCIR account needed; OCIR stays one flag away) |
| Isolation | Calico in policy-only mode, because OKE's VCN-native CNI does not enforce `NetworkPolicy` on its own |
| DNS | optional: a public zone in OCI DNS managed by ExternalDNS, records for every host, delegated from your registrar once |

## Prerequisites

- An OCI tenancy and the [`oci` CLI](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm) configured (`~/.oci/config`).
- [Terraform](https://developer.hashicorp.com/terraform/install) 1.5+ or [OpenTofu](https://opentofu.org), `kubectl`, an ssh key pair.
- A domain (or a subdomain of one) for the platform, for example `oci.example.com`. Its records can live in OCI DNS (recommended: shpyrd then manages them) or anywhere you can create a wildcard record.
- The `shpyrd` CLI ([Installation](/docs/installation)).

## 1. Network and cluster

```shell
git clone https://github.com/shpyrd-io/shpyrd
cd shpyrd/contrib/oci/terraform
cp terraform.tfvars.example terraform.tfvars
```

Fill in `terraform.tfvars`:

```hcl
tenancy_ocid = "ocid1.tenancy.oc1..aaaa"
region       = "sa-saopaulo-1"
name         = "shpyrd-prod"

cluster_type = "BASIC_CLUSTER"       # ENHANCED_CLUSTER for workload identity (per-cluster fee)
node_shape   = "VM.Standard.E5.Flex" # or VM.Standard.A1.Flex (Always Free, arm64) where the region has capacity
node_ocpus   = 2
node_memory_gb = 12
node_count   = 2

ssh_public_key_path = "~/.ssh/id_ed25519.pub"

dns_zone = "oci.example.com"   # public zone in OCI DNS; "" for none
dns_auth = "key"               # key (any cluster), workload (enhanced clusters), none
```

```shell
terraform init
terraform apply          # about 15 minutes
```

`terraform output` prints what the next steps need: the reserved load balancer address, the DNS zone's name servers, the private load balancer subnet, the DNS automation user and key, and the full `shpyrd cluster init` command (`terraform output next_steps`).

{% callout title="Basic or Enhanced?" %}
A **Basic** cluster has a free control plane; the DNS automation then uses an IAM user with an API key that Terraform creates (`dns_auth = "key"`). An **Enhanced** cluster costs about $0.10 per hour and adds workload identity: the cluster's service accounts get their permissions directly and no key exists anywhere (`dns_auth = "workload"`). Basic upgrades to Enhanced in place.
{% /callout %}

## 2. Reach the cluster

The API endpoint is private. Two scripts next to the Terraform handle it:

```shell
cd ..                    # contrib/oci
./kubeconfig.sh          # writes the kubectl context oke-<name>, pointed at the tunnel
./tunnel.sh &            # Bastion port-forwarding session + ssh tunnel on 127.0.0.1:6443
kubectl --context oke-shpyrd-prod get nodes
```

Bastion sessions live three hours; run `tunnel.sh` again when kubectl stops answering. The kubeconfig keeps TLS verification against the endpoint's own address.

## 3. Delegate the zone

With `dns_zone` set, Terraform created the zone in OCI DNS and the wildcard record pointing at the reserved address. Delegate it once from the parent zone at your registrar, with the name servers from `terraform output dns_zone_nameservers`:

```
oci  NS  ns1.p201.dns.oraclecloud.net
oci  NS  ns2.p201.dns.oraclecloud.net
oci  NS  ns3.p201.dns.oraclecloud.net
oci  NS  ns4.p201.dns.oraclecloud.net
```

From then on nothing in that zone is touched by hand: ExternalDNS publishes a record for every platform and project hostname. Without a zone in OCI DNS, create `*.oci.example.com  A  <reserved address>` wherever the domain lives; `cluster init` prints the record and waits for it.

## 4. Install the platform

The command `terraform output next_steps` printed, roughly:

```shell
shpyrd cluster init --context oke-shpyrd-prod --profile oci --domain oci.example.com \
  --set SHPYRD_ACME_EMAIL=you@example.com --set SHPYRD_LB_IP=<reserved address> \
  --dns oci --dns-compartment <compartment ocid> --dns-tenancy <tenancy ocid> --dns-region sa-saopaulo-1 \
  --dns-user <user ocid> --dns-key-file contrib/oci/terraform/shpyrd-prod-dns.pem \
  --internal-lb-subnet <private lb subnet ocid> \
  --enable auth-local
```

What happens, in order:

| Level | Components |
| --- | --- |
| rc0 | Prometheus Operator CRDs |
| rc1 | Calico (policy only), cert-manager, the registry credential |
| rc2 | Let's Encrypt issuers, the platform CA (generated in the cluster) and trust bundle, ingress-nginx (public load balancer on the reserved address) and, with a subnet, the internal one, the registry and the node trust for it, ExternalDNS and the OCI DNS-01 solver |
| rc3 | kpack with the Paketo builder (pushed to the in-cluster registry), kube-prometheus-stack, the wildcard certificate |
| rc4 | the shpyrd server |

The installer waits for the load balancer address, for `shpyrd.<domain>` to resolve on public resolvers, and for the certificates. Twenty minutes on a fresh cluster, most of it downloads and Let's Encrypt. The summary at the end:

```
  Dashboard:  https://shpyrd.oci.example.com
  Grafana:    https://grafana.oci.example.com
  Registry:   in-cluster at 10.96.0.50:5000 (TLS from the platform CA, credential in Secret shpyrd-registry)
  External LB:   147.15.59.84 (ExternalDNS: *.oci.example.com)
  Internal LB:   10.0.10.179 (ExternalDNS: per host, exposure:internal)
```

Everything you passed is recorded in the cluster: later runs (`brew upgrade shpyrd && shpyrd cluster init --context oke-shpyrd-prod --profile oci`) need no flags, and the key never leaves the Secrets the installer wrote.

## 5. First sign-in

A fresh cluster has one credential: the **admin token**, a Secret the installer generated. The CLI already uses it through your kubeconfig; for the dashboard, open it signed in through a one-time ticket, or copy the token into the sign-in page:

```shell
shpyrd cluster dashboard --context oke-shpyrd-prod    # opens https://shpyrd.oci.example.com signed in
shpyrd cluster token --context oke-shpyrd-prod        # prints the token for the "Admin token" field
```

The token is a shared, full-rights credential meant for bootstrap and automation. Give people their own accounts instead (`--enable auth-local` above installed the sign-in service), then put yourself in a platform-admin team - the moment the first team exists, roles are enforced and anyone without one sees nothing:

```shell
shpyrd users add you@example.com --name "You" --context oke-shpyrd-prod        # prompts for a password
shpyrd teams create platform --platform-role platform-admin --member you@example.com --context oke-shpyrd-prod
```

Sign in at `https://shpyrd.oci.example.com` with that email and password. When every administrator has an account, switch the token off: `shpyrd cluster token --disable` (the CLI keeps working through your kubeconfig; `--enable` turns it back on). Company sign-in (Okta, any OpenID Connect issuer, GitHub, Google) and teams mapped to identity provider groups are in [Extensions and sign-in](/docs/extensions) and [Teams, roles and security](/docs/access).

## 6. First project

```shell
shpyrd projects create shop --context oke-shpyrd-prod
shpyrd deploy --project shop --context oke-shpyrd-prod
```

Certificates are publicly trusted, so `https://shop.oci.example.com` opens with no warnings - from its first request when a DNS provider issues the wildcard. From here everything works as on the local profile: [Deploying](/docs/deploying), [Databases and caches](/docs/databases), [Domains and exposure](/docs/domains).

## Costs

At the defaults, on the pay-as-you-go price list: two `VM.Standard.E5.Flex` workers (2 OCPU, 12 GB) about $0.10 per hour each; a flexible load balancer at 10 Mbps; the registry's 50 GB block volume; Enhanced clusters add about $0.10 per hour. The Bastion service, the VCN, the reserved address, the DNS zone and Calico are free; DNS queries are billed per million. `VM.Standard.A1.Flex` (Ampere, arm64) is Always Free up to 4 OCPUs and 24 GB when the region has capacity - everything shpyrd runs is multi-arch.

## Good to know

- **Network policy.** OKE with VCN-native pod networking accepts `NetworkPolicy` objects without enforcing them. The profile installs Calico in policy-only mode (Oracle's supported path) so projects are isolated from each other and from the instance metadata service; `--set SHPYRD_NETWORK_POLICY=none` skips it on a cluster that already enforces policies. `cluster init` warns on any cluster where it finds no policy engine.
- **Block volumes start at 50 GB.** A `shpyrd volumes create data --size 1Gi` is created at 50Gi and the command says so; the registry's volume is 50 GB for that reason. Snapshots (`shpyrd volumes snapshot`) are block volume backups. See [Volumes on Oracle Cloud](/docs/resources#volumes-on-oracle-cloud).
- **Shared volumes need File Storage.** Set `shared_storage = true` in `terraform.tfvars` and pass the two `--set SHPYRD_FSS_MOUNT_TARGET=… --set SHPYRD_FSS_AD=…` values `next_steps` prints to `shpyrd cluster init`. It needs the File Storage service limits `Mount Target Count` and `File System Count` above zero in the availability domain (Console: Governance > Limits, Quotas and Usage > File Storage); some tenancies start at 0 and must request an increase. The mount target is free; file systems bill by the space used.
- **CRI-O.** OKE nodes run CRI-O, which refuses unqualified image names such as `redis:7`; shpyrd's own images are fully qualified, and so should yours be in a Dockerfile.
- **OCIR instead of the in-cluster registry.** `--registry-host <region>.ocir.io/<tenancy-namespace> --registry-user <namespace>/<user> --registry-token-file <file>` uses OCIR; the registry components are then skipped.
- **Upgrading.** `brew upgrade shpyrd` then `shpyrd cluster init` on the context. The recorded settings carry over; the CLI installs the server image of its own version.

## Tear down

Delete what Kubernetes created in the cloud first, or it outlives the cluster: `shpyrd projects destroy` for every project (their volumes go with them), then the load balancer Services.

```shell
kubectl --context oke-shpyrd-prod delete svc -n ingress-nginx ingress-nginx-controller
kubectl --context oke-shpyrd-prod delete svc -n ingress-nginx-internal ingress-nginx-internal-controller
kubectl --context oke-shpyrd-prod delete pvc -A --all
cd contrib/oci/terraform && terraform destroy
```

The DNS zone's delegation at the registrar is the one thing left to remove by hand.
