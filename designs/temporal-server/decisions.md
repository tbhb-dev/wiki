---
type: Design Appendix
title: Self-hosted Temporal server decisions
description: The open and decided questions of the self-hosted Temporal server design.
tags: [temporal, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Self-hosted Temporal server decisions

Back to [the design](README.md).

## Open

None.

## Decided

### D14. Does Temporal connect through PgBouncer?

- **Decided:** 2026-11-24, by ops.
- **Rule:** no. Temporal connects straight to its own Postgres. PgBouncer serves the shared Postgres instance only, and this design runs no PgBouncer exporter.
- **Why:** Temporal's own connection pools have fixed maximums in its persistence config, so the server never opens more connections than the instance allows.
- **Undo:** two-way door. Put a pooler in front later and point `POSTGRES_SEEDS` at it.
- **Record:** the pull request that adds this decision.

### D13. How much memory does each container get?

- **Decided:** 2026-11-24, by ops.
- **Rule:** Postgres 4G with 1G of shared buffers and 200 connections, the server 2G, Alloy 512M, the UI, the proxy, and the sidecar 256M each, the exporter 128M, and the one-shot jobs 256M. Nobody has measured these, and plan step 8 checks peak use against them.
- **Why:** limits keep Temporal from starving the shared Postgres instance on the same host.
- **Undo:** two-way door. Change a limit in a reviewed pull request.
- **Record:** the pull request that adds this decision.

### D12. Which versions run, and how are they pinned?

- **Decided:** 2026-11-24, by ops.
- **Rule:** the Temporal server and `admin-tools` images at 1.32.2, Postgres 16, the Postgres exporter at v0.21.1, the UI at its newest release at build, and nginx and Alloy at the versions the observability stack pins. Every image pins its digest at build. Renovate proposes bumps, and ops merges the server and `admin-tools` bumps together.
- **Undo:** two-way door, inside the upgrade rules of D5.
- **Record:** the pull request that adds this decision.

### D11. How is Temporal observed?

- **Decided:** 2026-11-24, by ops.
- **Rule:** an Alloy container in the project, with no scrape port outside it.
  - **Metrics:** Temporal's server metrics, the Postgres exporter, container metrics through cAdvisor, and the host's file system metrics for the Temporal volume go to Mimir over the OpenTelemetry Protocol (OTLP).
  - **Logs:** Postgres writes jsonlog files to the mounted log volume, Temporal logs to standard output, and Alloy sends both to Loki.
  - **Traces:** the server exports OpenTelemetry spans over OTLP to Tempo, through its `OTEL_` environment variables.
  - **Dashboard:** one Grafana dashboard, built live, then backported into the observability stack's dashboard files as the capacity dashboards were.
  - **Alerts:** instance down, volume fill, backup failure, connection saturation, persistence latency, and task backlog, routed through the existing notification policy.
  - **Health:** Compose health checks on every long-running container, which the deploy's health gate reads, as the observability role's gate does.
- **Why:** it matches the shared Postgres design and keeps the scrape inside the project network.
- **Undo:** two-way door. Remove the Alloy container and the dashboard.
- **Record:** the pull request that adds this decision.

### D10. Where do the credentials live?

- **Decided:** 2026-11-24, by ops for the vault and its items. The coordinating role decides the Connect token's grant under the operator's standing authorization of 2026-11-12, in its own decision entry.
- **Rule:** a new vault holds the Postgres passwords, the object store key, and the backup passphrase. The deploy resolves `op://` references into files mode 0600, and git holds only the references. Certificates come from the internal CA, never from 1Password.
- **Undo:** two-way door. Revoke the token's grant and drop the vault.
- **Record:** the pull request that adds this decision.

### D9. Where does the Temporal database live?

- **Decided:** 2026-11-24, by ops.
- **Rule:** a second Postgres instance on its own ext4 logical volume in the data host's volume group, beside the shared Postgres instance's volume. Free space stays unallocated for growth and snapshots. The private companion states sizes and paths.
- **Why:** a full shared Postgres volume, an upgrade or crash of the shared Postgres instance leaves Temporal running.
- **Undo:** two-way door until data grows. Dump the databases and restore them elsewhere.
- **Record:** the pull request that adds this decision.

### D8. How are the databases backed up?

- **Decided:** 2026-11-24, by ops.
- **Rule:** a nightly encrypted `pg_dump` in custom format of both databases, plus the roles, to Temporal's own bucket in the object store, keeping 7 daily and 4 weekly copies. A snapshot of the Temporal volume precedes every upgrade and goes away within 24 hours. A restore drill follows the first backup and every minor upgrade.
- **Why:** a dump restores into a fresh instance anywhere, and the object store runs on another host.
- **Undo:** two-way door. Stop the timer and delete the bucket.
- **Record:** the pull request that adds this decision.

### D7. Which namespaces exist, and who creates them?

- **Decided:** 2026-11-24, by ops.
- **Rule:** ops creates `workflows`, with 14 days of retention, and `sandbox`, with 1 day, through a one-shot job that skips a namespace that already exists. Archival stays off.
- **Undo:** two-way door. Change the retention, or delete a namespace.
- **Record:** the pull request that adds this decision.

### D6. How does the Web UI reach the operator?

- **Decided:** 2026-11-24, by ops.
- **Rule:** the UI answers at `temporal.tbhb.cloud` and at the tailnet label `temporal-ui`, through an nginx HTTPS proxy in the project and the UI Tailscale Service, the same pattern as Grafana. The DNS record goes through the usual DNS path. Write actions stay off, and the UI reaches the frontend with its own client certificate.
- **Why:** one pattern for every web UI, and a read-only UI behind a narrow grant leaves workflow changes to clients with certificates.
- **Undo:** two-way door. Remove the record and the service.
- **Record:** the pull request that adds this decision.

### D5. How do schema setup and upgrades run?

- **Decided:** 2026-11-24, by ops.
- **Rule:** the `init` profile runs `temporal-sql-tool create` and `setup-schema` once. Every deploy runs `update-schema` on both databases, and the server starts only after it exits with success. Upgrades move one minor version at a time, schema first.
- **Undo:** two-way door for the process. A schema change goes forward only, and the pre-upgrade snapshot of D8 is the way back.
- **Record:** the pull request that adds this decision.

### D4. Does Temporal run as one process or as separate services?

- **Decided:** 2026-11-24, by ops.
- **Rule:** one container runs all four services in one process.
- **Why:** the expected load of a few workflows an hour, an estimate nobody has measured, needs no split.
- **Undo:** two-way door. Start one container per service with `TEMPORAL_SERVICES`, against the same database.
- **Record:** the pull request that adds this decision.

### D3. How many history shards does the Temporal database get?

- **Decided:** 2026-11-24, by tbhb, through the coordination pane.
- **Rule:** 128 history shards, as recommended.
- **Why:** Temporal fixes the shard count when it first writes the database. The template's default of 4 suits a test server, and 128 leaves headroom for one small process.
- **Undo:** one-way door. A new count needs a new database and a drain of the old cluster's workflows.
- **Record:** the pull request that adds this decision.

### D2. Is mutual Transport Layer Security the only access control on the frontend?

- **Decided:** 2026-11-24, by tbhb, through the coordination pane.
- **Rule:** yes, as recommended, for now. The frontend accepts only client certificates from the internal certificate authority (CA) and runs without an authorizer. The CA gains Temporal's server name and client issuance for the gateway, the workers, the UI, and the server itself.
- **Why:** Temporal's built-in claim mapper reads JSON Web Tokens (JWT) from a key set, and nothing here issues them. Any client with a valid certificate gets full rights in every namespace, and the tailnet grant of D1 narrows who reaches the port.
- **Undo:** two-way door. Adding an authorizer later changes server configuration only, and clients keep their certificates.
- **Record:** the pull request that adds this decision.

### D1. May ops add a tailnet tag, two Tailscale Services, and their grants for Temporal?

- **Decided:** 2026-11-24, by tbhb, through the coordination pane.
- **Rule:** yes, as recommended. The tailnet policy on main already defines the frontend service on TCP 7233 and the UI service on TCP 443, with grants and tests for the operator and the operator's devices. This adds one new tag for the Temporal sidecar, approved as the host of both services, and grants with policy tests: the event gateway's host and the agent hosts reach the frontend, and the data host reaches the object store for backups.
- **Why:** the sidecar is the only way in, as for the CA and 1Password Connect apps. A new tag or grant needs the operator's decision before the policy change merges.
- **Undo:** two-way door. Revert the policy pull request, and the next apply removes the grants.
- **Record:** the pull request that adds this decision.
