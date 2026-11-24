---
type: Design Appendix
title: Self-hosted Temporal server alternatives
description: The options the Temporal server design didn't take, and why.
tags: [temporal, alternatives, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Self-hosted Temporal server alternatives

Back to [the design](README.md).

| Option | Why not |
| --- | --- |
| Temporal Cloud | A paid service, which the operator ruled out |
| The `temporalio/auto-setup` image | Docker Hub marks it unmaintained, and it doesn't get updates |
| Temporal's development server, `temporal server start-dev` | It targets local development, not a long-lived service |
| Separate frontend, history, matching, and worker containers | Four containers and their membership ports for a load one process serves. The same database supports a split later |
| A database inside the shared Postgres instance | An upgrade, crash, or full volume of the shared Postgres instance would stop Temporal too, and the two workloads would share one connection limit and one memory budget |
| Postgres 17, 18, or 19 | Temporal's documentation lists Postgres up to 16 as tested. A newer major waits for Temporal to list it |
| Elasticsearch or OpenSearch for visibility | Postgres 12 and later serves advanced visibility, and a search cluster adds a service to run and back up |
| Base backups with write-ahead log archiving | Point-in-time recovery adds a write-ahead log archive to run and test. ops can add it later without changing the dumps |
| A published host port for the frontend, as the object store does | With the sidecar, Compose publishes no port, so neither the host firewall nor the host's own tailnet node matters |
| The default JSON Web Token claim mapper | It needs a token issuer and a signing key that don't exist yet |
| Single sign-on for the Web UI | Nothing provides single sign-on yet, and the read-only UI behind a narrow grant serves one operator |
| The host's own collector scraping the containers | It needs a fixed subnet and host access to container addresses. An Alloy container in the project matches the shared Postgres design |
| PgBouncer in front of Temporal's Postgres | Temporal's own connection pools have fixed maximums, so a pooler adds a hop and a service for no gain. PgBouncer serves the shared Postgres instance only |
| A published host port bound to the tailnet address for the UI | The Tailscale Service and the proxy give the same names as Grafana with no host port |
