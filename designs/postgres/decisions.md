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

<!-- cspell:words basebackup textsearch -->

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
- **Rule:** Session pooling, with both extensions preloaded and `search_path` set per role.
- **Undo:** Set the pool mode to transaction after a spike shows AGE's queries work in it.
- **Record:** [A. Evidence](A-evidence.md).

### D2. Which Postgres major?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** Postgres 17, where AGE 1.7.0 and `pg_textsearch` v1.6.0 both apply. AGE 1.8.0 exists for Postgres 18 and 15 but not 17, and 17 has the longer AGE history, so 18 waits until a 1.8.0 build has run on 17 elsewhere or until 18 has a record of its own. Revisit at the first major upgrade.
- **Undo:** Rebuild the image on another major and restore from a dump.
- **Record:** [A. Evidence](A-evidence.md).

### D3. Which backup method?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** Nightly `pg_dump` in custom format, encrypted, uploaded to the object store. A base backup waits until a dump takes more than an hour. The hour is reasoning, not a measurement: it keeps the dump inside the quiet night window and a drill inside a working hour, and the first drill measures the real time.
- **Recovery after a switch:** Stop the stack, empty the data directory, unpack the newest base backup and its archived logs there, set the recovery signal file, start `postgres`, and confirm with the row-count check and one graph query.
- **Undo:** Switch the unit to `pg_basebackup`.
- **Record:** This design, as reasoning.

### D4. How do metrics and logs leave the host?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** A collector in the Compose project scrapes both exporters and reads container logs, then sends both to the observability stack's OTLP service. No port opens for scraping.
- **Undo:** Remove the collector and the exporter.
- **Record:** This design, as reasoning: the choice follows the observability stack's collectors and has no separate source record.
