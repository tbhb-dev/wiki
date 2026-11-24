---
type: Design Appendix
title: Graph Postgres decisions
description: The open and decided questions of the graph Postgres design.
tags: [graph-postgres, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words basebackup textsearch -->

# Graph Postgres decisions

Back to [the design](README.md).

## Open

### D1. May the tailnet policy gain a tag and grants for the graph database?

- **Recommendation:** Yes. Add one tag for the database's sidecar and one Tailscale Service for PgBouncer. Grant the operator's members, console devices, agent hosts, and lab nodes TCP 5432 to that Service, and don't grant the tag outbound access. Let the tag on the data host connect to the object store Service on TCP 443 for backups, which the policy withholds today. Test one `accept` for each source class and one `deny` for a runner and for the tag's own sidecar. The private companion specifies the tag and the Service.
- **Why:** The database is reachable only over the tailnet, and a new tag and grant widen who can connect.
- **If yes:** A policy pull request adds the tag and the grants along with `accept` and `deny` tests. The deploy waits for its merge.
- **If no:** The database binds only to loopback and every client connects through an SSH tunnel, which the agent hosts can't use unattended.
- **Reversible:** Yes. Removing the tag closes the access.

## Decided

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
- **Rule:** Nightly `pg_dump` in custom format, encrypted, uploaded to the object store. A base backup waits until a dump takes more than an hour. The hour is reasoning, not a measurement: it keeps the dump inside the scheduled night window and a drill inside a working hour, and the first drill measures the real time.
- **Recovery after a switch:** Stop the stack and empty the data directory. Unpack the newest base backup and its archived logs there, then set the recovery signal file. Start `postgres` and confirm with the row-count check and one graph query.
- **Undo:** Switch the unit to `pg_basebackup`.
- **Record:** This design, as reasoning.

### D4. How do metrics and logs leave the host?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** A collector in the Compose project scrapes both exporters and reads container logs, then sends both to the observability stack's OTLP service. No port opens for scraping.
- **Undo:** Remove the collector and the exporter.
- **Record:** This design, as reasoning: the choice follows the collectors in the observability stack and has no separate source record.
