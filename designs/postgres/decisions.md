---
type: Design Appendix
title: Shared Postgres decisions
description: The open and decided questions of the shared Postgres design.
tags: [postgres, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words basebackup dumpall journald textsearch -->

# Shared Postgres decisions

Back to [the design](README.md).

## Open

None.

## Decided

### D1. May the tailnet policy gain a tag and grants for the shared Postgres server?

- **Decided:** 2026-11-24, by the operator, as recommended.
- **Rule:** Add one tag for the server's sidecar and one Tailscale Service for PgBouncer. Grant the operator's members, console devices, agent hosts, and lab nodes TCP 5432 to that Service, and give the tag no outbound grants. Let the data host's tag reach the object store Service on TCP 443 for backups, which the policy withholds today. Test one `accept` for each source class and one `deny` for a runner and for the tag's own sidecar. The private companion names the tag and the Service.
- **Why:** The server is reachable only over the tailnet, and a new tag and grant widen who can connect.
- **Next:** A policy pull request adds the tag, the grants, and `accept` and `deny` tests, and the deploy waits for its merge.
- **Undo:** Remove the tag, which closes the access.
- **Record:** This design.

### D6. One shared server, with a database and logins for each client?

- **Decided:** 2026-11-24, by the operator.
- **Rule:** One shared production Postgres server serves every service that doesn't get a dedicated server. Each client gets its own database and logins, so the operator can revoke one client or login class without touching another. The context graph is the first client.
- **Undo:** Move a client to a dedicated server with a dump and restore.
- **Record:** This design.

### D5. Which PgBouncer pool mode?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** Session pooling for Apache A Graph Extension (AGE), with the server preloading both extensions and setting `search_path` per role. Whether this removes per-session setup and whether the intended client's SQL-level `PREPARE` queries work through session pooling stay unverified until the first build test.
- **Undo:** Set the pool mode to transaction after a spike shows AGE's queries work in it.
- **Record:** [A. Evidence](A-evidence.md).

### D2. Which Postgres major?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** Postgres 17, where AGE 1.7.0 and `pg_textsearch` v1.6.0 both apply. AGE 1.8.0 exists for Postgres 18 and 15 but not 17, and 17 has the longer AGE history, so 18 waits until AGE publishes a 1.8.0 build for 17, or until a build on 18 has run the first-test checks. Revisit at the first major upgrade.
- **Undo:** Rebuild the image on another major and restore from a dump.
- **Record:** [A. Evidence](A-evidence.md).

### D3. Which backup method?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** A nightly `pg_dump -Fc` runs once per database as a backup role with `CONNECT` on each database and the `pg_read_all_data` role, and the superuser runs `pg_dumpall --roles-only` over the local socket. The unit encrypts the archive before upload to the object store, and the backup encryption key stays off the data host (the private companion names it); the roles dump contains Salted Challenge Response Authentication Mechanism (SCRAM) verifiers. A restore applies the roles file first, then runs `pg_restore --create` for each database so database-level grants and settings return, then installs current SCRAM verifiers from the credential store.
- **Switch threshold:** A base backup waits until a dump takes more than an hour. The hour is reasoning, not a measurement: it keeps the dump inside the quiet night window and a drill inside a working hour, and the first drill measures the real time.
- **Recovery after a switch:** Stop the stack, empty the data directory, unpack the newest base backup and its archived logs there, set the recovery signal file, start `postgres`, and confirm with the row-count check and one graph query.
- **Undo:** Switch the unit to `pg_basebackup`.
- **Record:** This design, as reasoning.

### D4. How do metrics and logs leave the host?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** Containers use the `journald` logging driver, and a collector in the Compose project scrapes both exporters and reads only this project's container logs through Alloy's journal source. The collector mounts only the host journal directory read-only, runs as a non-root user in the journal group with no capabilities and no Docker socket, and sends metrics and logs to the observability stack's OpenTelemetry Protocol (OTLP) service; this avoids exposing other containers' configuration files. OTLP and backup upload leave through the data host's own tailnet identity, not the sidecar's, which has no outbound grants.
- **Undo:** Remove the collector and the exporter.
- **Record:** This design, as reasoning: the choice follows the observability stack's collectors and has no separate source record.
