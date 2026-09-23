---
title: Databases and caches
description: PostgreSQL databases (CloudNativePG) and Redis-compatible stores (Valkey) as project resources, attached to apps as config vars.
---

Two extensions add data stores to projects: `postgres` runs PostgreSQL databases with the CloudNativePG operator, `redis` runs Valkey or Redis caches and queues. Attach one to the app and it appears as `DATABASE_URL` or `REDIS_URL`. {% .lead %}

```shell
shpyrd extensions enable postgres
shpyrd extensions enable redis
```

## PostgreSQL

```shell
shpyrd pg create db --project shop                        # PostgreSQL 17, 5Gi, 1 instance, default size
shpyrd pg create db --project shop --size shared-m --storage 20Gi --instances 3   # HA with replicas
shpyrd pg list --project shop
shpyrd pg psql db --project shop -- -c 'select version()'
shpyrd pg delete db --project shop --yes                  # refused while attached (or --force)
```

Each database is its own [CloudNativePG](https://cloudnative-pg.io) cluster in the project namespace: streaming replication and failover when `--instances` is 2 or 3, a `db-rw` service for the primary and `db-ro` for replicas, a database `app` owned by user `app`. The instance size sets CPU and memory, with a floor of 256 MiB because PostgreSQL does not start below it; storage grows (`shpyrd pg create` again is not needed, edit the resource) but never shrinks.

Not there yet: backups to object storage and point-in-time recovery, connection pooling and credential rotation ([RFC-0009](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0009-postgres-resource.md)).

## Redis and Valkey

```shell
shpyrd redis create cache --project shop                  # Valkey 8, cache mode
shpyrd redis create queue --project shop --persistent --storage 2Gi   # append-only file on a volume
shpyrd redis create legacy --project shop --engine redis  # upstream Redis 7
shpyrd redis cli cache --project shop -- INFO memory
```

[Valkey](https://valkey.io) (BSD licensed, protocol compatible) is the default engine; `--engine redis` selects upstream Redis. A store is a single instance run by the shpyrd controller: `maxmemory` is 75% of the size's memory; a **cache** evicts with `allkeys-lru` and loses its content on restart, which is the expected behaviour of a cache; a **persistent** store keeps an append-only file on a volume and refuses writes instead of evicting when full. Persistence cannot change after creation. High availability through an operator comes later ([RFC-0010](https://github.com/shpyrd-io/shpyrd/blob/main/rfcs/0010-redis-resource.md)).

## Attaching

```shell
shpyrd attach db --project shop                 # DATABASE_URL, DATABASE_HOST, DATABASE_PORT, DATABASE_USER, DATABASE_PASSWORD, DATABASE_NAME
shpyrd attach cache --project shop              # REDIS_URL, REDIS_HOST, REDIS_PORT, REDIS_PASSWORD
shpyrd attach sessions --prefix SESSIONS        # SESSIONS_URL, ... when two stores of the same kind are attached
shpyrd detach db --project shop
```

Attaching adds a binding to the app and releases it (`Attach Postgres db`); the variables are read-only in the Config tab and in `shpyrd secrets list`, shown with the resource providing them, and they win over a config var of the same name. If the resource is still provisioning, the app waits (phase `Pending`, "waiting for an attached resource") and releases when it is ready. Detaching removes the variables in a new release, and rollback restores the attachments a release had. The dashboard's Resources card has **Attach**/**Detach** buttons and an **Add resource** menu with the same forms.

Resources live inside the project's network policy: only the project's own processes can reach them, other projects cannot.
