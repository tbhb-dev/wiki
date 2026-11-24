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

<!-- cspell:words cutover dumpall gosu pgbouncer textsearch -->

# Graph Postgres for the context graph

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | the ops repository's agent | 2026-11-24 | 1 open decision in [decisions.md](decisions.md), about 2 minutes |

## Summary

The context graph needs a store that runs graph queries and ranked text search. This design proposes one Postgres 17 container on the data host, with Apache AGE for graph queries and `pg_textsearch` for ranked search, behind a PgBouncer container.

Clients connect only to PgBouncer over the tailnet. The database uses one logical volume on the data volume group, a nightly encrypted dump, and credentials from 1Password Connect. Metrics and logs flow to the observability stack.

The one thing it asks of the operator is a tailnet tag and grants for the sidecar. A private companion contains the deployment specifics.

## Proposal

One Compose project, `graph-postgres`, runs on the data host. It follows the object store and certificate authority projects: pinned image digests, one Compose network, no published host port, and a Tailscale sidecar.

| Component | What it is | State |
| --- | --- | --- |
| `postgres` | Postgres 17 built from the official image, with AGE and `pg_textsearch` added, run as the image's `postgres` user | Data directory on its own logical volume |
| `pgbouncer` | PgBouncer in session pooling mode, the only container the sidecar exposes | Rendered config and auth file |
| `postgres-exporter`, `pgbouncer-exporter` | Prometheus community exporters with read-only roles | None |
| `collector` | One Grafana Alloy that scrapes both exporters and reads the container log files, then sends both to the observability stack | None |
| `ts` | Tailscale sidecar that serves PgBouncer as a Tailscale Service | Sidecar state directory |
| backup unit | A systemd timer that runs `pg_dump` and `pg_dumpall --roles-only`. It encrypts both and uploads them | Archive directory |

Each Postgres instance on the data host gets one logical volume on the data volume group, formatted ext4 and mounted under the host's service directory, with free space left for growth and snapshots. The private companion at the same path, `designs/graph-postgres/`, gives sizes, names, and paths.

## How it works

### Image and versions

Postgres 17 is the major where a published AGE release and `pg_textsearch` both apply, and AGE provides 1.6.0 and 1.7.0 for it. The AGE port to Postgres 18 has only its first release, so 18 waits (appendix A). A short Dockerfile starts from the official image by digest and adds AGE 1.7.0 for Postgres 17 and the `pg_textsearch` release. The Compose file pins the built image by tag and digest, and a Renovate rule raises each bump as its own pull request.

### PgBouncer and the pool mode

Session pooling is the recommendation. AGE needs `LOAD 'age'` and a `search_path` change on every connection, and SQL-level `PREPARE` statements, which AGE's parameterized queries use, stay on one server connection only in session mode (appendix A). PgBouncer's reset query runs after each client in session mode and doesn't run in transaction mode.

The server preloads both extensions, and `ALTER ROLE` sets `search_path` for each client role to remove the per-session setup. Because Postgres 17 doesn't report `search_path`, PgBouncer applies only the startup value. The reset query in session mode clears any client change. Transaction pooling is a later spike, because the client count today is small and session pooling already caps server connections.

### Backup and restore

A nightly `pg_dump` in custom format and a `pg_dumpall --roles-only` file run from a one-shot container, because `pg_dump` leaves out cluster-wide roles. The unit encrypts both and writes checksums before uploading them to the object store. It keeps seven daily and four weekly archives. A rebuild from git and the ledger is the last-resort restore.

A restore into a fresh container applies the roles file first, then runs `pg_restore`, then the deploy installs each role's current SCRAM verifier from the credential store. A monthly restore drill follows those steps with the newest archive in a scratch container on the data host, and runs a row-count check and one graph query. It runs again after each major upgrade.

### Upgrades

Minor releases of Postgres, AGE, `pg_textsearch`, PgBouncer, and the exporters go through the image build and a pull request. A major upgrade means a new volume, a dump, and a restore into the new image. The old volume stays until the drill passes.

### Limits

Containers have memory and process limits, drop all capabilities, and gain no new privileges. The official Postgres image starts as root and drops to the `postgres` user with `gosu`, which needs capabilities that `cap_drop: ALL` removes. The `postgres` container starts directly as the image's `postgres` user, which owns the data directory. The entrypoint skips `gosu` and doesn't need any capability. A deploy check starts the container and reads `pg_up`. Appendix A gives first values and a connection budget to adjust after a week of metrics.

## How clients connect

| Question | Recommendation |
| --- | --- |
| Which clients? | The `tbhbdev graph` command on agent hosts and the operator's laptop, and the future indexer service. Agents use the command, never a database client. |
| Network path? | The tailnet only. A Tailscale Service, like Grafana's, fronts PgBouncer. Postgres has no route except the project network. |
| Transport encryption? | The tailnet's WireGuard layer for now. Database-level encryption with a certificate from the internal certificate authority waits for the first client that leaves the tailnet. |
| Authentication? | SCRAM passwords from Connect, one per role. The tailnet grant is the first gate and the password is the second. Client certificates come later. |
| Roles per client? | Each client class has its own read-only login: the command on agent hosts, and the operator. The indexer has a read-write login. A separate owner login runs migrations from the deploy and PgBouncer never lists it. One login per class lets the operator revoke a class and tell classes apart in logs. A login per host is a later step. The private companion specifies the logins. |
| How does a client find the address and credentials? | The `tbhbdev` configuration names the Service and a Connect item reference. The command reads the password from Connect at run time and never stores it. |

## Observability

| Signal | Recommendation |
| --- | --- |
| Metrics | The project's collector scrapes `postgres_exporter` and `pgbouncer_exporter`. The host's collector supplies container and volume metrics. Both send metrics to Mimir. |
| Logs | Postgres logs in JSON, PgBouncer logs as text. The collector mounts the Docker log directory read-only and reads each container's `json-file` log with Alloy's file source. After parsing the Postgres JSON lines in a processing stage, it sends both through the OTLP exporter into Loki. |
| Traces | None. Clients don't trace database calls yet, and the exporters give the same signal. |
| Dashboards | Build one Grafana dashboard per service live and then backport it to the repository, following the capacity dashboards. |
| Alerts | Instance down, volume usage exceeds 85 percent, backup failure or a missed night, and connection saturation. They use the existing Grafana notification policy. |
| Health checks | A container health check on each service, and the deploy's health gate reads them before it reports success. |

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| AGE names its tags `-rc0`, and GitHub marks the 1.7.0 builds as releases | A defect in graph queries | Pin 1.7.0 for Postgres 17, and the build's tests check AGE queries | Dates and flags in appendix A |
| The context-graph design chose SQLite | This store contradicts a decided record | The context-graph design owner revises it before the build starts | Existing: that design's decided record |
| AGE's setup under transaction pooling | Wrong results or errors | Session pooling now, with a spike before any change | Existing: appendix A |
| One disk stores the volume | A disk failure loses the data | Nightly dump to the object store and the rebuild path | New |

## Plan

| Step | Change | Done when | Owner |
| --- | --- | --- | --- |
| 1 | This design and its private companion merge | The operator answers D1 and two reviews clear each exact head | the ops repository's agent |
| 2 | The project directory in the ops repository: Dockerfile, Compose file, backup unit, tests, dashboards, and alert rules | Its pull request's `conclusion` check is green at the head | the ops repository's agent |
| 3 | The volume and the deploy from merged code | Every service reports healthy and `pg_up` reads 1 in Mimir | the ops repository's agent |
| 4 | The first restore drill | The drill's log shows a matching row count | the ops repository's agent |

## Later

- The graph and search schema, the backfill from the current store, and the cutover. This design covers the store only.
- Transaction pooling, after a spike on AGE's prepared queries.
- Database-level encryption and client certificates.
- JSON logs from PgBouncer, a base backup if a dump outgrows an hour, a login per host, and a replica.

## Appendices

- [Decisions](decisions.md): 1 open, then the decided record.
- [A. Evidence](A-evidence.md) gives version sources and pooling behavior, plus first limits.
- A private appendix contains the host-specific details for this design.
