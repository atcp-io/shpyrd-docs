---
title: Architecture guide
description: Internals and architecture overview for Shpyrd
---

Shpyrd is a opinionated platform that facilitates application lifecycle on Kubernetes

---

## Components

- Ingress nginx, connected to a loadbalancer (paas or metal based), services are routed through this service mesh
- Cert manager, issues certificate for local domains ( apps.yourdomain ) and public domains (apps.yourdomain.com)
- Local Registry, a volatile secure local docker for image storage
- Trust manager, distribute certificate toolchain to service
- External DNS, manages dns entries on services like Route53 and Cloudflare
- Kustomize, uses templates to build deploy configs
- Kpack, build image based on a repo using native buildback (like heroku), uploads to Local Registry.
- Permissions and Roles, TBD
- Grafana Loki, logs, multiple storage backends (ex: s3)
- Promtail, Log Collector
- Prometheus and Statsd Redirector, Metrics Collection
- Jaeger, Open Telemetry

---

### Controller Component

It's responsible for making sure the managed kubernetes is configured correctly and all components are up to date. The controller keep watching git repositories and build images and deploy on detected changes.

#### POC

Temporary using FluxCD with Kustomize Watch on Github (to be replaced by shpyrd controller):
- FluxCD watches github repository for Kustomize (native k8s yml files), and then apply on change
- FluxCD watches new build on Docker, and change the image version on deployment (k8s)

### Service Component

It's the backend for the web UI service. Is where operators and developers setup new app environments, schedule jobs and monitors the app lifecycle.