---
type: Design Draft
title: Shared Postgres server
description: tbhb-dev's shared production Postgres server for any service without a dedicated one, with the Apache A Graph Extension (AGE) and pg_textsearch extensions and PgBouncer in front, on the data host. The context graph is its first client.
tags: [postgres, pgbouncer, shared-server, design]
status: draft
owner: the ops repository's agent
updated: "2026-11-24"
decisions_open: 0
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words cutover dumpall gosu journald pgbouncer textsearch UID userspace -->

# Shared Postgres server

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | the ops repository's agent | 2026-11-24 | Nothing |

## Summary

tbhb-dev needs one shared production Postgres server for any service that needs Postgres and doesn't get a dedicated server, as the workflow engine does. This design proposes one Postgres 17 container on the data host behind a PgBouncer container, with a database and logins for each client. The server installs Apache A Graph Extension (AGE) and `pg_textsearch`, and the context graph is the first client, using AGE for graph queries and `pg_textsearch` for ranked search.

Clients reach PgBouncer over the tailnet and nothing else. The database uses one logical volume on the data volume group, nightly encrypted dumps, one per database, and credentials from 1Password Connect. Metrics and logs flow to the observability stack.

A private companion contains the deployment specifics.

## Proposal

One Compose project, `postgres`, runs on the data host. It follows the object store and certificate authority projects: pinned image digests, no published host port, and a Tailscale sidecar.

The project uses two Compose networks: an internal network for Postgres, PgBouncer, the Postgres exporter, and the collector, and a second network for PgBouncer, the PgBouncer exporter, the sidecar, and the collector. The collector scrapes both exporters, and the sidecar serves only PgBouncer.

| Component | What it is | State |
| --- | --- | --- |
| `postgres` | Postgres 17 built from the official image, with AGE and `pg_textsearch` added, run as the image's `postgres` user | Data directory on its own logical volume |
| `pgbouncer` | PgBouncer in session pooling mode, the only container the sidecar exposes | Rendered config and auth file |
| `postgres-exporter`, `pgbouncer-exporter` | Prometheus community exporters with read-only roles | None |
| `collector` | One Grafana Alloy that scrapes both exporters and reads this project's container logs from the host journal, then sends both to the observability stack | None |
| `ts` | Tailscale sidecar that serves PgBouncer as a Tailscale Service, with userspace networking and no added capability | Sidecar state directory |
| backup unit | A systemd timer that runs `pg_dump -Fc` once per database and `pg_dumpall --roles-only`, encrypts the archive, and uploads it | Archive directory |

Each Postgres instance on the data host gets one logical volume on the data volume group, formatted ext4 and mounted under the host's service directory, with free space left for growth and snapshots. The private companion at the same path, `designs/postgres/`, gives sizes, names, and paths.

## How it works

### Image and versions

AGE 1.7.0 and `pg_textsearch` both support 17 and 18, and 17 has the wider AGE history, so Postgres 17 is the conservative pick (appendix A). A short Dockerfile starts from the official image by digest and adds AGE 1.7.0 for Postgres 17 and the `pg_textsearch` release. The Compose file pins the built image by tag and digest, and a Renovate rule raises each bump as its own pull request.

The PgBouncer image must be 1.26.1 or later, because 1.17.0 through 1.26.0 have a Salted Challenge Response Authentication Mechanism (SCRAM) startup flaw that 1.26.1 fixes (appendix A).

### PgBouncer and the pool mode

Session pooling is the recommendation. AGE needs `LOAD 'age'` and a `search_path` change on every connection; whether the intended client's AGE queries use SQL-level `PREPARE` and work through session pooling stays unverified until the first build test (appendix A). PgBouncer's reset query runs after each client in session mode and doesn't run in transaction mode.

The server preloads both extensions, and `ALTER ROLE` sets `search_path` for each client role; whether these settings remove the per-session setup stays unverified until the first build test. Postgres 17 doesn't report `search_path`, so PgBouncer applies only the startup value, and session mode's reset query clears any client change. Transaction pooling is a later spike, because the client count today is small and session pooling already caps server connections.

### Backup and restore

A nightly `pg_dump -Fc` runs once per database from a one-shot container as a backup role with `CONNECT` on each database and the `pg_read_all_data` role. The superuser runs `pg_dumpall --roles-only` over the local socket, because `pg_dump` leaves out cluster-wide roles. The unit encrypts the archive before upload, writes checksums, and uploads it to the object store.

The unit keeps seven daily and four weekly archives, and the encryption key stays off the data host; the private companion names it. A rebuild from git and the ledger is the last-resort restore.

A restore into a fresh container applies the roles file first, then runs `pg_restore --create` for each database so database-level grants and settings return, then the deploy installs each role's current SCRAM verifier from the credential store. A monthly restore drill follows those steps with the newest archive in a scratch container on the data host, and runs a row-count check and one graph query. It runs again after each major upgrade.

### Upgrades

Minor releases of Postgres, AGE, `pg_textsearch`, PgBouncer, and the exporters go through the image build and a pull request. A major upgrade means a new volume, a dump, and a restore into the new image. The old volume stays until the drill passes.

### Limits

Every container has a memory limit, a process limit, no capabilities, and no new privileges. The official Postgres image starts as root and drops to the `postgres` user with `gosu`, which needs capabilities that `cap_drop: ALL` removes. The deploy sets the volume's owner to the image's `postgres` user identifier (UID) before first start, and the container starts directly as that user, so the entrypoint skips `gosu` and needs no capability.

A deploy check starts the container and reads `pg_up`. Appendix A gives first values and a connection budget to adjust after a week of metrics.

## How clients connect

| Question | Recommendation |
| --- | --- |
| Which clients? | Any service that needs Postgres and has no dedicated server. The first is the context graph, through the `tbhbdev graph` command on agent hosts and the operator's laptop and a future indexer service. Agents use that command, never a database client. |
| Network path? | The tailnet only. A Tailscale Service, like Grafana's, fronts PgBouncer. Postgres has no route except the internal network. |
| Transport encryption? | The tailnet's WireGuard layer for now. Database-level encryption with a certificate from the internal certificate authority waits for the first client that leaves the tailnet. |
| Authentication? | SCRAM passwords from Connect, one per role. The tailnet grant is the first gate and the password is the second. Client certificates come later. |
| Databases and roles per client? | Each client gets its own database and its own logins, by design: a read-only login per client class, a read-write login for a service, and an owner login for migrations that PgBouncer never lists. The deploy creates the extensions only in a database that needs them. One login per class lets the operator revoke a class and tell classes apart in logs; the private companion names the first client's logins. |
| How does a client find the address and credentials? | The `tbhbdev` configuration names the Service and a Connect item reference. The command reads the password from Connect at run time and never stores it. |

Migrations run from a one-off container on the internal network as the owner login.

## Observability

| Signal | Recommendation |
| --- | --- |
| Metrics | `postgres_exporter` and `pgbouncer_exporter` scraped by the project's collector, plus container and volume metrics from the host's collector, all into Mimir. |
| Logs | Containers use the `journald` logging driver; Postgres logs in JSON, and PgBouncer logs as text. The collector mounts only the host journal directory read-only and uses Alloy's journal source filtered to this project's containers, which avoids exposing other containers' configuration files. It runs as a non-root user in the journal group, with no capabilities and no Docker socket, parses the Postgres JSON lines, and sends both log streams through the OpenTelemetry Protocol (OTLP) exporter into Loki. |
| Traces | None. No client traces database calls yet, and the exporters give the same signal. |
| Dashboards | One Grafana dashboard per service, built live and then backported to the repository, as the capacity dashboards were. |
| Alerts | Instance down, volume above 85 percent, backup failure or a missed night, and connection saturation. They use the existing Grafana notification policy. |
| Health checks | A container health check on each service, and the deploy's health gate reads them before it reports success. |

OTLP and backup upload leave through the data host's own tailnet identity, not the sidecar's, which has no outbound grants.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| AGE names its tags `-rc0`, and GitHub marks the 1.7.0 builds as releases | A defect in graph queries | Pin 1.7.0 for Postgres 17, and the build's tests check AGE queries | Dates and flags in appendix A |
| The context-graph design chose SQLite | The first client contradicts a decided record | The context-graph design owner revises it before the build starts | Existing: that design's decided record |
| AGE's setup under transaction pooling | Wrong results or errors | Session pooling now, with a spike before any change | Existing: appendix A |
| One disk holds the volume; the roles dump contains SCRAM verifiers | A disk failure loses the data; archive or encryption-key exposure compromises credentials | Encrypt the archive before upload and keep the backup encryption key off the data host (the private companion names it); use nightly dumps to the object store and the rebuild path | New |
| AGE graphs restore poorly from a dump | A restore loses graph data or behavior | The restore drill covers a graph, and a major upgrade uses a dump and restore test first | Existing: known weak spot |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | This design and its private companion merge | Two reviews clear each exact head and the policy pull request merges | the ops repository's agent |
| 2 | The project directory in the ops repository: Dockerfile, Compose file, backup unit, tests, dashboards, and alert rules | Its pull request's `conclusion` check is green at the head | the ops repository's agent |
| 3 | The volume and the deploy from merged code | Every service reports healthy and `pg_up` reads 1 in Mimir | the ops repository's agent |
| 4 | The first restore drill | The drill's log shows a matching row count | the ops repository's agent |

## Later

- The context graph's schema, the backfill from its current store, and the cutover. This design covers the server only.
- Transaction pooling, after a spike on AGE's prepared queries.
- Database-level encryption and client certificates.
- JSON logs from PgBouncer, a base backup if a dump outgrows an hour, a login per host, and a replica.

## Appendices

- [Decisions](decisions.md): 0 open, then the decided record.
- [A. Evidence](A-evidence.md): version sources, pooling behavior, and first limits.
- A private appendix holds the host-specific details for this design.
