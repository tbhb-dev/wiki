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

# Graph Postgres decisions

Back to [the design](README.md).

## Open

### D1. May the tailnet policy gain a tag and grants for the graph database?

- **Recommendation:** Yes. Add one tag for the database's sidecar, with access for the server and the console devices on the Postgres port, and no outbound grants.
- **Why:** The database is reachable only over the tailnet, and a new tag and grant widen who can connect.
- **If yes:** A policy pull request adds the tag, the grants, and `accept` and `deny` tests, and the deploy waits for its merge.
- **If no:** The database binds only to loopback and every client reaches it through an SSH tunnel, which the agent hosts can't use unattended.
- **Reversible:** Yes. Removing the tag closes the access.

## Decided

### D2. Which Postgres major?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** Postgres 18, the one major that AGE and pg_textsearch both support today.
- **Undo:** Rebuild the image on another major and restore from a dump.
- **Record:** [A. Evidence](A-evidence.md).

### D3. Which backup method?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** Nightly `pg_dump` in custom format, encrypted, uploaded to the object store. A base backup waits until the data outgrows a dump that finishes in an hour.
- **Undo:** Switch the unit to `pg_basebackup`.
- **Record:** This design.

### D4. How do metrics and logs leave the host?

- **Decided:** 2026-11-24, by the ops repository's agent.
- **Rule:** A collector in the Compose project scrapes the exporter and reads container logs, then sends both to the observability stack's OTLP service. No port opens for scraping.
- **Undo:** Remove the collector and the exporter.
- **Record:** This design.
