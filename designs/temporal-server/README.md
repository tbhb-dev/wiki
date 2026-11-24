---
type: Design Draft
title: Self-hosted Temporal server
description: A Temporal server with its own Postgres instance on the data host, for the event gateway and the workflow workers.
tags: [temporal, postgres, deployment, design]
status: draft
owner: ops
updated: "2026-11-24"
decisions_open: 0
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Self-hosted Temporal server

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | ops | 2026-11-24 | Nothing |

## Summary

The operator chose Temporal to run event-driven workflows behind the event gateway, self-hosted and with no paid service. The gateway's Temporal target needs a server, and none runs yet.

This design covers what deploys today: one Temporal server process runs all four services on the data host, with its own Postgres 16 instance on its own logical volume. A Tailscale Service fronts the remote procedure call frontend, and every client presents a certificate from the internal certificate authority under mutual Transport Layer Security (TLS). The web user interface (UI) answers at `temporal.tbhb.cloud` through Hypertext Transfer Protocol Secure (HTTPS), behind the same proxy pattern as Grafana, while an Alloy collector in the project sends metrics, logs, and traces to the observability host.

The operator answered D1, D2, and D3: the new tailnet tag with its services and grants, mutual TLS as the only access control, and the history shard count, which Temporal fixes for the life of the database. Zero decisions remain open.

Deployment specifics, such as host names, volume names, paths, tags, and 1Password item names, live in the private companion at `designs/temporal-server` in the private wiki repository.

## Proposal

The data host runs one Docker Compose project, `temporal`, from files ops reviewed and merged. None of its containers publishes a host port:

| Container | Image | Runs |
| --- | --- | --- |
| `postgres` | `postgres:16`, the newest 16 minor at build | The databases `temporal` and `temporal_visibility`, with JSON logs |
| `schema` | `temporalio/admin-tools`, same version as the server | `temporal-sql-tool update-schema` once per deploy, then exits |
| `temporal` | `temporalio/server:1.32.2` | `temporal-server start` with the frontend, history, matching, and worker services in one process |
| `ui` | `temporalio/ui`, the newest release at build | The Temporal Web UI, read-only |
| `ui-tls` | `nginxinc/nginx-unprivileged`, the nginx version the observability stack pins | The HTTPS proxy in front of the UI, as Grafana's `grafana-tls` |
| `postgres-exporter` | `prometheuscommunity/postgres-exporter:v0.21.1` | Postgres metrics |
| `alloy` | `grafana/alloy`, the version the observability stack pins | Scrapes and log collection, sent over OpenTelemetry Protocol (OTLP) |
| `ts` | `tailscale/tailscale`, the version the other apps pin | The sidecar that hosts the frontend and UI Tailscale Services |

Every image pins its digest at build, and Renovate proposes bumps. Two profiles hold the one-shot jobs: `init` creates the databases on the first deploy, and `backup` runs the nightly dump.

The roles:

- **ops** owns the host, the volume, the Compose project, the deploy, the backups, the dashboards, and the alert rules. It runs every deploy from merged code.
- **The DNS maintainer** adds the `temporal.tbhb.cloud` record through the usual DNS path.
- **The administrator role** sets up the Tailscale Services after the policy change merges.
- **The gateway maintainer** points the gateway's Temporal target at this server with a client certificate ([event gateway](../event-gateway/README.md)).
- **tools** runs the workflow workers, which connect the same way.
- **The operator** answered D1, D2, and D3 on 2026-11-24.

## How it works

### One process, one database instance

One process runs all four services, because the expected load of a few workflows an hour needs no split ([D4](decisions.md#d4-does-temporal-run-as-one-process-or-as-separate-services)). Temporal gets its own Postgres instance, a second one beside the shared Postgres instance on the data host, so neither one's load, upgrade, or crash touches the other ([D9](decisions.md#d9-where-does-the-temporal-database-live)). Temporal tests Postgres up to 16, so this instance runs 16 ([appendix A](A-evidence.md#versions)).

The data host's second disk is one volume group, with one logical volume per Postgres instance and free space for growth and snapshots. The private companion states the sizes and paths. The data directory sits one level below the mount point, so an unmounted volume makes Postgres fail rather than start empty.

Temporal connects straight to its own Postgres, with no PgBouncer in between ([D14](decisions.md#d14-does-temporal-connect-through-pgbouncer)).

### How clients connect

| Question | Answer |
| --- | --- |
| Who connects | The event gateway, the tools workers, the `temporal` command-line tool, and the Web UI |
| Network path | A Tailscale Service fronts the gRPC frontend on TCP 7233 and forwards raw TCP to the server. The UI reaches the frontend on the Compose network |
| Name | The frontend Tailscale Service's name |
| Transport security | Mutual TLS. Each client holds a leaf from the internal certificate authority (step-ca), and the server checks it ([D2](decisions.md#d2-is-mutual-transport-layer-security-the-only-access-control-on-the-frontend)) |
| Namespace authorization | None today. Any valid client certificate has full rights in every namespace, and the tailnet grant narrows who reaches the port ([D1](decisions.md#d1-may-ops-add-a-tailnet-tag-two-tailscale-services-and-their-grants-for-temporal)) |
| Discovery | Each client reads the frontend address and namespace from its own configuration. The `tbhbdev` configuration carries them for agents, and the CA's issuance path delivers the certificate. No secret reaches a client through 1Password |
| Database | Only the server and the jobs reach Postgres, with Salted Challenge Response Authentication Mechanism (SCRAM) passwords from 1Password Connect |

### The Web UI

The UI answers only at `temporal.tbhb.cloud` over HTTPS, as Grafana answers at `grafana.tbhb.cloud` ([D6](decisions.md#d6-how-does-the-web-ui-reach-the-operator)). An `ui-tls` nginx container serves HTTPS on container port 8443 with a host-provided certificate that covers `temporal.tbhb.cloud`, the pattern the observability stack's `grafana-tls` proxy uses. The UI Tailscale Service forwards Transmission Control Protocol (TCP) port 443 to it, the Domain Name System (DNS) record for `temporal.tbhb.cloud` points at that service, and the UI runs with write actions off.

### Namespaces and schema

ops creates `workflows` with 14 days of retention and `sandbox` with 1 day, through a one-shot job ([D7](decisions.md#d7-which-namespaces-exist-and-who-creates-them)). The first deploy runs the `init` profile once. Every later deploy runs `schema` first, and the server starts only after it exits with success. Upgrades move one minor version at a time ([D5](decisions.md#d5-how-do-schema-setup-and-upgrades-run)).

### Observability

An Alloy container in the project scrapes Temporal's metrics endpoint, the Postgres exporter, and container metrics, reads Postgres jsonlog files from the mounted log volume and Temporal's container logs, and sends everything over the existing OTLP path to Mimir and Loki. Temporal's traces go to Tempo over OTLP. One Grafana dashboard covers the service, and alert rules route like the existing ones ([D11](decisions.md#d11-how-is-temporal-observed)). [Appendix C](C-deployment.md#observability) lists the metrics, alerts, and health checks.

### Backups

A nightly `pg_dump` of both databases, encrypted, goes to Temporal's own bucket in the object store on another host. A snapshot of the Temporal volume precedes every upgrade ([D8](decisions.md#d8-how-are-the-databases-backed-up)).

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| The CA can't issue client certificates today | No client can connect under mutual TLS | Plan step 2 changes the CA before the deploy | Existing, the CA's own documentation |
| The client-auth setting covers internode traffic too | The built-in worker service fails to reach its own frontend | Give the server its own client certificate, or mount a full config file | Unverified until the deploy |
| CA leaves expire after 72 hours by default | Clients and the server fail mutual TLS within days | A renewal timer on each host, and an alert at 24 hours left | Existing lifetime, unverified renewal path |
| Every client certificate has full rights | A leaked worker key can reset or stop any workflow | Short certificate lifetimes, the tailnet grant, and a later claim mapper | New |
| One disk holds both Postgres instances | A disk failure loses both databases | Nightly dumps leave the host | New |
| A snapshot fills the volume group's free space | The snapshot becomes invalid, and the shared Postgres volume can't grow | Snapshot size cap, removal within 24 hours, and a fill alert | New |

## Plan

| Step | What changes | Done when | Owner |
| --- | --- | --- | --- |
| 1 | A pull request to the tailnet policy: the new tag as host of the existing frontend and UI services, the grants of [D1](decisions.md#d1-may-ops-add-a-tailnet-tag-two-tailscale-services-and-their-grants-for-temporal), and policy tests | The policy tests accept each granted path and deny the rest | ops |
| 2 | CA change: the server name and client issuance for the gateway, workers, UI, and server | A test certificate for each client verifies against the internal root | ops |
| 3 | 1Password items in a new vault and the Connect token grant | The deploy's dry run resolves every reference | ops, with the grant from the coordinating role |
| 4 | Host change: the Temporal logical volume through the existing volume role, with ledger rows | The volume shows its size and ext4 at its mount | ops |
| 5 | `apps/temporal`, built by a Codex build run: Compose file, proxy and sidecar config, Alloy config, dynamic config, backup job, dashboard, and offline tests | The repository's gates pass and two reviews clear the exact head | ops |
| 6 | Tailscale Services for the frontend and the UI, after step 1 merges | Both services list the sidecar as their host | Administrator role |
| 7 | DNS record for `temporal.tbhb.cloud` through the usual DNS path, with an internal DNS alias to the UI's tailnet service name, which the private companion sets | The name resolves on the tailnet to the UI service | DNS maintainer |
| 8 | Deploy from merged main: `init` once, then `up`, then the namespace job | The health gate passes, rerunning the schema job succeeds without changes, and `temporal operator cluster health` succeeds through the frontend service | ops |
| 9 | Certificate renewal timers, the backup timer, alert rules, and the first restore drill | The drill's scratch server lists the workflows of the restored dump | ops |
| 10 | The gateway's Temporal target connects | Event gateway plan step 11b passes against this server | Gateway maintainer |

## Later

These wait for a need, and nothing in today's deploy depends on them:

- A claim mapper that reads each certificate's identity and grants per-namespace roles, as a custom server build.
- Single sign-on for the UI, and UI write actions.
- Archival to the object store.
- Separate service containers, if load grows.
- Write-ahead log archiving for point-in-time recovery.
- A copy of the dumps on the backup host, once the object store replicates.

## Appendices

- [Decisions](decisions.md): 0 open, then the decided record.
- [A. Evidence](A-evidence.md): the sources for each claim, labeled proven, existing, new, or unverified.
- [B. Alternatives considered](B-alternatives.md): the options not taken, and why.
- [C. Deployment](C-deployment.md): the Compose sketch, ports, credentials, observability, and backups.
