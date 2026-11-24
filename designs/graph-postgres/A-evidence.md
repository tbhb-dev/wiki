---
type: Design Appendix
title: Graph Postgres evidence
description: Sources and labels for the claims in the graph Postgres design.
tags: [graph-postgres, evidence, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words pgbouncer -->

# Graph Postgres evidence

Back to [the design](README.md).

## Versions

### Support matrix

Read on 2026-11-24.

| Component | Version | Supported majors | Source |
| --- | --- | --- | --- |
| Apache AGE | 1.7.0, tags `PG17/v1.7.0-rc0` and `PG18/v1.7.0-rc0` | 17 and 18, one tag per major | <https://github.com/apache/age/releases> |
| Apache AGE | 1.6.0 | 14, 15, 16, and 17 | <https://github.com/apache/age/releases> |
| Apache AGE | 1.8.0, tags for 15 and 18 | 15 (marked pre-release) and 18 | <https://github.com/apache/age/releases> |
| `pg_textsearch` | v1.6.0 | 17, 18, and 19 | <https://github.com/timescale/pg_textsearch/releases> |
| `postgres_exporter` | v0.21.1 | Not stated | <https://github.com/prometheus-community/postgres_exporter/releases> |
| PgBouncer | 1.26.1 | Not stated | <https://www.pgbouncer.org/changelog.html> |
| `pgbouncer_exporter` | v0.12.1 | PgBouncer 1.24 and later | <https://github.com/prometheus-community/pgbouncer_exporter/releases> |

### Claims

| Claim | Label | Source |
| --- | --- | --- |
| AGE's README lists Postgres 11 through 18 as supported | Existing | <https://github.com/apache/age> |
| AGE names every tag `-rc0`, but GitHub marks the 1.7.0 builds as releases (`prerelease` false): Postgres 17 published 2026-02-11 and Postgres 18 published 2026-01-21. Only the 1.8.0 build for 15, published 2026-09-19, is a pre-release. Read 2026-11-24 with the GitHub releases API | Existing | <https://github.com/apache/age/releases> |
| The 1.7.0 notes for 18 say no earlier AGE exists for 18, so it has no upgrade path | Existing | <https://github.com/apache/age/releases/tag/PG18%2Fv1.7.0-rc0> |
| The overlap of AGE 1.7.0 and `pg_textsearch` is Postgres 17 and 18, and 17 has the older and wider AGE history | New | The matrix |

## Pooling

Read on 2026-11-24 at <https://www.pgbouncer.org/config.html>, <https://www.pgbouncer.org/changelog.html>, and <https://age.apache.org/age-manual/master/intro/setup.html>.

| Claim | Label | Source |
| --- | --- | --- |
| AGE needs `LOAD 'age';` on every connection and `SET search_path = ag_catalog, "$user", public;` for short queries | Existing | The AGE setup page |
| The default pool mode is session, and a server returns to the pool when the client disconnects | Existing | The PgBouncer config page |
| `server_reset_query` defaults to `DISCARD ALL` and doesn't run in transaction pooling unless `server_reset_query_always` is on | Existing | The PgBouncer config page |
| PgBouncer tracks protocol-level prepared statements in transaction mode, and forwards SQL-level `PREPARE` and `EXECUTE` unchanged | Existing | The PgBouncer config page |
| PgBouncer tracks `search_path` by default on Postgres 18 and later, and applies only the startup value before that | Existing | The PgBouncer config page and 1.26.0 changelog |
| The latest PgBouncer is 1.26.1 from 2026-10-14, which fixes a SCRAM startup flaw in 1.17.0 through 1.26.0 | Existing | The PgBouncer changelog |
| `pgbouncer_exporter` is at v0.12.1 from 2026-06-26 and supports PgBouncer 1.24 and later | Existing | <https://github.com/prometheus-community/pgbouncer_exporter/releases> |
| Preloading both extensions makes `LOAD` unnecessary | Unverified | The build's first test checks it |
| AGE's parameterized queries need SQL-level `PREPARE` | Unverified | The build's first test checks it |

The build confirms the exact tags and image names before it pins digests.

## First limits

Start with 16 GiB of memory for `postgres`, 256 MiB for PgBouncer and each exporter, 384 MiB for the collector, and a process limit of 1,024. Set `shared_buffers` to 4 GiB. These numbers are new and unverified until a week of metrics exists.

## Existing patterns

The Compose layout, digest pins, hardening, tailnet sidecar, and OTLP wiring follow the object store and certificate authority projects and the observability stack in the ops repository. The restore drill follows the certificate authority's drill.
