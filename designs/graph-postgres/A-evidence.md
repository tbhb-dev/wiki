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

# Graph Postgres evidence

Back to [the design](README.md).

## Versions

Read on 2026-11-24.

| Claim | Label | Source |
| --- | --- | --- |
| AGE has releases for Postgres 18: 1.7.0 and a 1.8.0 candidate, tagged `PG18/v1.8.0-rc0` | Existing | <https://github.com/apache/age/releases> |
| AGE 1.8.0 appears for Postgres 15 and 18 only, and 1.7.0 for Postgres 17 and 18 | Existing | <https://github.com/apache/age/releases> |
| pg_textsearch supports Postgres 17, 18, and 19, and its latest release is v1.6.0 | Existing | <https://github.com/timescale/pg_textsearch> and its releases page |
| pg_textsearch needs `shared_preload_libraries` and `CREATE EXTENSION` per database | Existing | <https://github.com/timescale/pg_textsearch> |
| The overlap of both extensions is Postgres 17 and 18, and 18 is the newer | New | The three rows above |
| `postgres_exporter` is at v0.21.1 and publishes images to the GitHub container registry | Existing | <https://github.com/prometheus-community/postgres_exporter/releases> |
| The release pages show day and month only, so the AGE release year is unverified | Unverified | The same pages |

The build confirms the exact tags and image names before it pins digests.

## First limits

Start with 16 GiB of memory for `postgres`, 256 MiB for the exporter, 384 MiB for the collector, and a process limit of 1,024. Set `shared_buffers` to 4 GiB. These numbers are new and unverified until a week of metrics exists.

## Existing patterns

The Compose layout, digest pins, hardening, tailnet sidecar, and OTLP wiring follow the object store and certificate authority projects and the observability stack in the ops repository. The restore drill follows the certificate authority's drill.
