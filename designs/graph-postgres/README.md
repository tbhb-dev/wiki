---
type: Design Draft
title: Graph Postgres for the context graph
description: A self-hosted Postgres container with the Apache AGE graph extension and pg_textsearch, on the data host, that stores the context graph.
tags: [graph-postgres, postgres, context-graph, design]
status: draft
owner: the ops repository's agent
updated: "2026-11-24"
decisions_open: 1
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Graph Postgres for the context graph

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | the ops repository's agent | 2026-11-24 | 1 open decision in [decisions.md](decisions.md), about 2 minutes |

## Summary

The context graph needs a store that runs graph queries and ranked text search at a size SQLite handles poorly. This design proposes one Postgres 18 container on the data host, with Apache AGE for graph queries and pg_textsearch for ranked search.

The container's data directory sits on a dedicated logical volume. A nightly encrypted dump goes to the object store, and a restore drill proves it. Credentials come from 1Password Connect, and metrics and logs reach the observability stack.

The one thing it asks of the operator is a tailnet tag and grants for the container.

## Proposal

One Compose project, `graph-postgres`, runs on the data host from `/srv/graph-postgres`. It follows the pattern of the object store and certificate authority projects: pinned image digests, one Compose network, no published host port, and a Tailscale sidecar that exposes the database on the tailnet only.

| Component | What it is | State |
| --- | --- | --- |
| `postgres` | Postgres 18 image built from the official image, with AGE and pg_textsearch added | Data directory on the `pg-graph` volume |
| `exporter` | The Prometheus community `postgres_exporter`, read-only role | None |
| `collector` | One Grafana Alloy that scrapes the exporter and reads container logs, then sends both over the observability stack's OTLP service | None |
| `ts` | Tailscale sidecar that serves Postgres on the tailnet | `./ts-state` |
| backup unit | A systemd timer that runs `pg_dump` and writes an encrypted archive | Archive directory on the host |

The volume layout also covers the other database on the data host:

| Volume | Size | Mount | Use |
| --- | --- | --- | --- |
| `pg-graph` | 2.5 TiB | `/srv/graph-postgres` | This design's data |
| `pg-temporal` | 500 GiB | `/srv/temporal-postgres` | The workflow engine's database |
| Free space in the volume group | about 640 GiB | none | Growth and LVM snapshots |

Each volume is ext4 on the data volume group, which holds the second disk. A private appendix holds host names and addresses.

## How it works

### Image and versions

Postgres 18 is the one major both extensions support today (appendix A). A short Dockerfile in the ops repository starts from the official `postgres:18` image by digest, installs the AGE release for Postgres 18, and installs the pg_textsearch release. The built image carries a pinned tag and a digest in the Compose file. A Renovate rule raises each bump as its own pull request.

### Roles and access

Four roles exist. `graph_owner` owns the schema and runs migrations. `graph_app` reads and writes through the server. `graph_ro` reads only. `exporter` has the `pg_monitor` role and nothing else. Connections use scram-sha-256 over the tailnet, with the tailnet identity as the first gate and the password as the second. Passwords sit in 1Password and reach the container as files that Connect renders at deploy.

### Backup and restore

A nightly `pg_dump` in custom format runs from a one-shot container, then the unit encrypts it, writes a checksum, and uploads the pair to the object store. It keeps seven daily and four weekly archives. The graph is a projection of git and the ledger, so a rebuild is the last-resort restore and the dump saves time.

A restore drill loads the newest archive into a scratch container on the data host and runs a row-count check and one graph query. The drill runs monthly and after each major upgrade.

### Upgrades

Minor releases of Postgres, AGE, and pg_textsearch go through the image build and a pull request. A major upgrade means a new volume, a dump, and a restore into the new image, with the old volume kept until the drill passes.

### Limits

The container gets a memory limit, a process limit, and the hardening pair the object store uses: all capabilities dropped and no new privileges. Appendix A gives the first numbers, to adjust after the first week of metrics.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| AGE for Postgres 18 is a release candidate tag on the releases page | A defect in graph queries | Pin 1.7.0 for Postgres 18 unless the 1.8.0 candidate has a final tag at build time | Unverified: appendix A |
| The context graph design chose SQLite | This store contradicts a decided record | The context-graph design owner revises it before the build starts | Existing: that design's decided record |
| The volume sits on one disk | A disk failure loses the data | Nightly dump to the object store and the rebuild path | New |
| pg_textsearch is new | Index bugs | The extension is optional for the graph, and a rebuild recreates the index | Unverified |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | This design merges | Two reviews clear the exact head and the operator answers D1 | the ops repository's agent |
| 2 | `apps/graph-postgres` in the ops repository with the Dockerfile, Compose file, backup unit, and tests | Its pull request's `conclusion` check is green at the head | the ops repository's agent |
| 3 | The `pg-graph` volume and the deploy from merged code | `docker compose ps` shows the services healthy and the exporter's `pg_up` reads 1 in the metrics store | the ops repository's agent |
| 4 | The first restore drill | The drill's log shows a matching row count | the ops repository's agent |

## Appendices

- [Decisions](decisions.md): 1 open, then the decided record.
- [A. Evidence](A-evidence.md): version sources, the first limits, and the labels for each claim.
- A private appendix holds the host-specific details for this design.
