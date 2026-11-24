---
type: Design Draft
title: Graph Postgres for the context graph
description: A self-hosted Postgres container with the Apache AGE graph extension and pg_textsearch, behind PgBouncer on the data host, that stores the context graph.
tags: [graph-postgres, postgres, pgbouncer, context-graph, design]
status: draft
owner: the ops repository's agent
updated: "2026-11-24"
decisions_open: 1
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words pgbouncer -->

# Graph Postgres for the context graph

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | the ops repository's agent | 2026-11-24 | 1 open decision in [decisions.md](decisions.md), about 2 minutes |

## Summary

The context graph needs a store that runs graph queries and ranked text search. This design proposes one Postgres 17 container on the data host, with Apache AGE for graph queries and `pg_textsearch` for ranked search, behind a PgBouncer container.

Clients reach PgBouncer over the tailnet and nothing else. The database uses one logical volume on the data volume group, a nightly encrypted dump, and credentials from 1Password Connect. Metrics and logs flow to the observability stack.

The one thing it asks of the operator is a tailnet tag and grants for the sidecar. A private companion holds the deployment specifics.

## Proposal

One Compose project, `graph-postgres`, runs on the data host. It follows the object store and certificate authority projects: pinned image digests, one Compose network, no published host port, and a Tailscale sidecar.

| Component | What it is | State |
| --- | --- | --- |
| `postgres` | Postgres 17 built from the official image, with AGE and `pg_textsearch` added | Data directory on its own logical volume |
| `pgbouncer` | PgBouncer in session pooling mode, the only container the sidecar exposes | Rendered config and auth file |
| `postgres-exporter`, `pgbouncer-exporter` | Prometheus community exporters with read-only roles | None |
| `collector` | One Grafana Alloy that scrapes both exporters and reads container logs, then sends both to the observability stack | None |
| `ts` | Tailscale sidecar that serves PgBouncer as a Tailscale Service | Sidecar state directory |
| backup unit | A systemd timer that runs `pg_dump`, encrypts it, and uploads it | Archive directory |

Each Postgres instance on the data host gets one logical volume on the data volume group, formatted ext4 and mounted under the host's service directory, with free space left for growth and snapshots. The private companion at the same path, `designs/graph-postgres/`, gives sizes, names, and paths.

## How it works

### Image and versions

Postgres 17 is the major where a published AGE release and `pg_textsearch` both apply, and AGE ships 1.6.0 and 1.7.0 for it. The AGE port to Postgres 18 has only its first release, so 18 waits (appendix A). A short Dockerfile starts from the official image by digest and adds AGE 1.7.0 for Postgres 17 and the `pg_textsearch` release. The Compose file pins the built image by tag and digest, and a Renovate rule raises each bump as its own pull request.

### PgBouncer and the pool mode

Session pooling is the recommendation. AGE needs `LOAD 'age'` and a `search_path` change on every connection, and SQL-level `PREPARE` statements, which AGE's parameterized queries use, stay on one server connection only in session mode (appendix A). PgBouncer's reset query runs after each client in session mode and doesn't run in transaction mode.

Two settings remove the per-session setup. The server preloads both extensions, and `ALTER ROLE` sets `search_path` for each client role. Postgres 17 doesn't report `search_path`, so PgBouncer applies only the startup value, and session mode's reset query clears any client change. Transaction pooling is a later spike, because the client count today is small and session pooling already caps server connections.

### Backup and restore

A nightly `pg_dump` in custom format runs from a one-shot container, then the unit encrypts it, writes a checksum, and uploads the pair to the object store. It keeps seven daily and four weekly archives. A rebuild from git and the ledger is the last-resort restore.

A monthly restore drill loads the newest archive into a scratch container on the data host and runs a row-count check and one graph query. It runs again after each major upgrade.

### Upgrades

Minor releases of Postgres, AGE, `pg_textsearch`, PgBouncer, and the exporters go through the image build and a pull request. A major upgrade means a new volume, a dump, and a restore into the new image. The old volume stays until the drill passes.

### Limits

Every container has a memory limit, a process limit, all capabilities dropped, and no new privileges. Appendix A gives first values to adjust after a week of metrics.

## How clients connect

| Question | Recommendation |
| --- | --- |
| Which clients? | The `tbhbdev graph` command on agent hosts and the operator's laptop, and the future indexer service. Agents use the command, never a database client. |
| Network path? | The tailnet only. A Tailscale Service, like Grafana's, fronts PgBouncer. Postgres has no route except the project network. |
| Transport encryption? | The tailnet's WireGuard layer for now. Database-level encryption with a certificate from the internal certificate authority waits for the first client that leaves the tailnet. |
| Authentication? | SCRAM passwords from Connect, one per role. The tailnet grant is the first gate and the password is the second. Client certificates come later. |
| Roles per client? | The command and the operator read as `graph_ro`. The indexer reads and writes as `graph_app`. `graph_owner` runs migrations from the deploy and PgBouncer never lists it. |
| How does a client find the address and credentials? | The `tbhbdev` configuration names the Service and a Connect item reference. The command reads the password from Connect at run time and never stores it. |

## Observability

| Signal | Recommendation |
| --- | --- |
| Metrics | `postgres_exporter` and `pgbouncer_exporter` scraped by the project's collector, plus container and volume metrics from the host's collector, all into Mimir. |
| Logs | Postgres logs in JSON, PgBouncer logs as text, through the collector into Loki. |
| Traces | None. No client traces database calls yet, and the exporters give the same signal. |
| Dashboards | One Grafana dashboard per service, built live and then backported to the repository, as the capacity dashboards were. |
| Alerts | Instance down, volume above 85 percent, backup failure or a missed night, and connection saturation. They use the existing Grafana notification policy. |
| Health checks | A container health check on each service, and the deploy's health gate reads them before it reports success. |

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| Every AGE tag ends in `-rc0`, so release status rests on GitHub's labels | A defect in graph queries | Pin 1.7.0 for Postgres 17, and the build's tests check AGE queries | Unverified: appendix A |
| The context-graph design chose SQLite | This store contradicts a decided record | The context-graph design owner revises it before the build starts | Existing: that design's decided record |
| AGE's setup under transaction pooling | Wrong results or errors | Session pooling now, with a spike before any change | Existing: appendix A |
| One disk holds the volume | A disk failure loses the data | Nightly dump to the object store and the rebuild path | New |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | This design and its private companion merge | Two reviews clear each exact head and the operator answers D1 | the ops repository's agent |
| 2 | The project directory in the ops repository: Dockerfile, Compose file, backup unit, tests, dashboards, and alert rules | Its pull request's `conclusion` check is green at the head | the ops repository's agent |
| 3 | The volume and the deploy from merged code | Every service reports healthy and `pg_up` reads 1 in Mimir | the ops repository's agent |
| 4 | The first restore drill | The drill's log shows a matching row count | the ops repository's agent |

## Later

- Transaction pooling, after a spike on AGE's prepared queries.
- Database-level encryption and client certificates.
- JSON logs from PgBouncer, a base backup if a dump outgrows an hour, and a replica.

## Appendices

- [Decisions](decisions.md): 1 open, then the decided record.
- [A. Evidence](A-evidence.md): version sources, pooling behavior, and first limits.
- A private appendix holds the host-specific details for this design.
