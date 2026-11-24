---
type: Design Appendix
title: Self-hosted Temporal server evidence
description: The sources behind each claim of the Temporal server design, labeled proven, existing, new, or unverified.
tags: [temporal, evidence, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Self-hosted Temporal server evidence

Back to [the design](README.md).

The author read every source on 2026-11-24. Nothing ran against a Temporal server, so no claim here carries the proven label. The labels follow [WRITING.md](../WRITING.md): **existing** means a primary source says so today, **new** means this design proposes it, and **unverified** means no source or run confirms it yet.

## Versions

- **Existing.** temporalio/temporal's newest releases are v1.32.2, v1.31.5, and v1.30.9, all published on 2026-11-02 ([releases](https://github.com/temporalio/temporal/releases)).
- **Existing.** Docker Hub's API lists `temporalio/server:1.32.2` at digest `sha256:7bf69188a6232fea14869b9cef3d94aa82a1601d99517f7e86c71f22da4e0670` and `temporalio/admin-tools:1.32.2` at `sha256:4a95866efba29d7e17ab012df5232795e8b0ad0f7d5f9d1ec4df321f7792a754`. Step 5 pins whatever the build reads, since a tag can move ([server tags](https://hub.docker.com/r/temporalio/server/tags), [`admin-tools` tags](https://hub.docker.com/r/temporalio/admin-tools/tags)).
- **Existing.** `temporalio/ui` has 2.58.1 as `latest`, published 2026-11-06, and Temporal's sample Compose files pair server 1.32.2 with UI 2.56.1 ([UI tags](https://hub.docker.com/r/temporalio/ui/tags), [samples `.env`](https://github.com/temporalio/samples-server/blob/main/compose/.env)).
- **Existing.** Temporal's persistence page lists Postgres 13.18, 14.15, 15.10, and 16.6 as tested, and doesn't name a newer major ([persistence](https://docs.temporal.io/temporal-service/persistence)).
- **Existing.** Postgres 16's current minor is 16.16, supported until 2028-11-09. Postgres 19.0 came out on 2026-11-19 ([versioning policy](https://www.postgresql.org/support/versioning/)).
- **Existing.** `postgres_exporter` v0.21.1 is the newest release tag on Docker Hub, and its README lists Postgres 13 to 18 as tested in its continuous integration ([exporter tags](https://hub.docker.com/r/prometheuscommunity/postgres-exporter/tags), [README at v0.21.1](https://github.com/prometheus-community/postgres_exporter/blob/v0.21.1/README.md)).

## Images and process

- **Existing.** The `temporalio/auto-setup` page says nobody maintains the image and it doesn't get updates, and points to `temporalio/server` and `temporalio/admin-tools` ([auto-setup](https://hub.docker.com/r/temporalio/auto-setup)).
- **Existing.** Temporal archived the temporalio/docker-compose repository and moved its examples to temporalio/samples-server ([docker-compose](https://github.com/temporalio/docker-compose)).
- **Existing.** Temporal's deployment guide says to use `temporalio/server` in production, to manage schema updates with `temporalio/admin-tools`, and not to expose Temporal hosts to the open internet ([deployment](https://docs.temporal.io/self-hosted-guide/deployment)).
- **Existing.** The server image's entrypoint runs `temporal-server start` with no service flag, and the start command defaults to the frontend, history, matching, and worker services, overridden by `--service` or `TEMPORAL_SERVICES` ([entrypoint.sh](https://github.com/temporalio/temporal/blob/v1.32.2/docker/scripts/sh/entrypoint.sh), [main.go](https://github.com/temporalio/temporal/blob/v1.32.2/cmd/server/main.go), [server.go](https://github.com/temporalio/temporal/blob/v1.32.2/temporal/server.go)).
- **Existing.** The server image runs as user `temporal`, UID 1000, on Alpine, and the `admin-tools` image carries `temporal`, `temporal-sql-tool`, `tdbg`, and the schema files under `/etc/temporal/schema` ([server Dockerfile](https://github.com/temporalio/temporal/blob/v1.32.2/docker/targets/server.Dockerfile), [`admin-tools` Dockerfile](https://github.com/temporalio/temporal/blob/v1.32.2/docker/targets/admin-tools.Dockerfile)).
- **Unverified.** All four services in one process share one Prometheus listener, and each service opens its own persistence connection pools.
- **Unverified.** The load estimate of a few workflows an hour. It comes from the review queue's expected volume, which nothing has measured.
- **Unverified.** The shard count limits how many workflow histories the history service updates at once, and 128 shards fit one small process. Temporal's server page says only that the count can't change.
- **Unverified.** The development server, `temporal server start-dev`, suits local tests only. The author read no source for it.

## Configuration

These come from the template embedded in the v1.32.2 binary ([config_template_embedded.yaml](https://github.com/temporalio/temporal/blob/v1.32.2/common/config/config_template_embedded.yaml)), unless a line names another source.

- **Existing.** `DB=postgres12` selects the Postgres plugin, with `POSTGRES_SEEDS`, `DB_PORT`, `POSTGRES_USER`, and `POSTGRES_PWD`. The password comes from the environment only. Pools default to 20 connections for the main store and 10 for visibility, set by `SQL_MAX_CONNS` and `SQL_VIS_MAX_CONNS`.
- **Existing.** `NUM_HISTORY_SHARDS` defaults to 4. Temporal's server page says the History Shard count can't change once set ([Temporal Server](https://docs.temporal.io/temporal-service/temporal-server)).
- **Existing.** `PROMETHEUS_ENDPOINT` turns on the Prometheus reporter at that address with a histogram timer type. The handler path defaults to `/metrics` ([configuration reference](https://docs.temporal.io/references/configuration)).
- **Existing.** Frontend gRPC defaults to 7233 and its HTTP port to 7243. History, matching, and worker take 7234, 7235, and 7239, with membership ports 6933 to 6939.
- **Existing.** `TEMPORAL_TLS_REQUIRE_CLIENT_AUTH` sets `requireClientAuth` in both the internode and the frontend server blocks, and `TEMPORAL_TLS_REFRESH_INTERVAL` reloads certificates.
- **Existing.** The template turns archival on at the cluster level with a file store, and sets each namespace's default to off.
- **Existing.** `TEMPORAL_AUTH_AUTHORIZER` and `TEMPORAL_AUTH_CLAIM_MAPPER` accept an empty value or `default`. Without an authorizer the server logs a warning, unless the flag `--allow-no-auth` or the variable `TEMPORAL_ALLOW_NO_AUTH` turns it off ([main.go](https://github.com/temporalio/temporal/blob/v1.32.2/cmd/server/main.go), [loader.go](https://github.com/temporalio/temporal/blob/v1.32.2/common/config/loader.go)).
- **Existing.** The security guide says the no-op authorizer allows every request, that the default claim mapper reads JSON Web Tokens from key set endpoints, and that per-namespace control needs the authorizer and claim mapper plug-ins ([security](https://docs.temporal.io/self-hosted-guide/security)).
- **Unverified.** A claim mapper that reads the certificate's identity needs a custom server build. The guide shows one only through Go code that calls `temporal.NewServer`.
- **Existing.** The UI reads `TEMPORAL_ADDRESS`, `TEMPORAL_TLS_CA`, `TEMPORAL_TLS_CERT`, `TEMPORAL_TLS_KEY`, `TEMPORAL_TLS_SERVER_NAME`, `TEMPORAL_DISABLE_WRITE_ACTIONS`, and `TEMPORAL_AUTH_ENABLED`, and listens on 8080 by default. Temporal's UI variables page returned 404, so this cites the UI's own config ([docker.yaml at v2.56.1](https://github.com/temporalio/ui/blob/v2.56.1/server/config/docker.yaml)).

## Schema and upgrades

- **Existing.** Temporal's Postgres setup script runs `temporal-sql-tool --plugin postgres12` with `create`, `setup-schema -v 0.0`, and `update-schema -d /etc/temporal/schema/postgresql/v12/<db>/versioned` for `temporal` and `temporal_visibility` ([setup-postgres.sh](https://github.com/temporalio/samples-server/blob/main/compose/scripts/setup-postgres.sh)).
- **Existing.** The upgrade guide says to update the schema before the binary, one minor version at a time, and to roll back the binary, never the schema. An older binary of an adjacent minor starts against a newer schema ([upgrade](https://docs.temporal.io/self-hosted-guide/upgrade-server)).
- **Unverified.** `update-schema` on a database already at the binary's version exits with success and changes nothing. Step 7 checks it.
- **Unverified.** `temporal operator namespace update` changes a namespace's retention, and `temporal operator cluster health` prints `SERVING` on a healthy frontend. Temporal's samples show only `temporal operator namespace create --retention` ([samples README](https://github.com/temporalio/samples-server/blob/main/compose/README.md)).

## Postgres

- **Existing.** `pg_dump` makes consistent backups while the database is in use and doesn't block readers or writers. It dumps one database, and `pg_dumpall` covers roles ([`pg_dump`](https://www.postgresql.org/docs/16/app-pgdump.html)).
- **Existing.** A frozen snapshot of the volume that holds the whole data directory, write-ahead log included, works while the server runs, and the restored copy replays the log as after a crash. A `CHECKPOINT` first shortens that replay ([file system backup](https://www.postgresql.org/docs/16/backup-file.html)).
- **Existing.** Postgres advises a data directory one level below a secondary volume's mount point, so that a missing volume fails cleanly ([creating a cluster](https://www.postgresql.org/docs/16/creating-cluster.html)).
- **Existing.** Temporal lists Postgres 12 and later for advanced visibility on server 1.20 and later ([visibility](https://docs.temporal.io/self-hosted-guide/visibility)).
- **Unverified.** A custom-format dump restores into an instance of the same or a newer Postgres major.
- **Unverified.** The official `postgres:16` image's `postgres` user has UID 999. The image's default entrypoint starts as root and switches to that user, so it needs capabilities that `cap_drop: [ALL]` removes and fails to start. The image documents starting it with `--user` set to the data directory's owner, which skips the switch and works with every capability dropped ([Docker Official Image](https://hub.docker.com/_/postgres), section "Arbitrary --user Notice").

## Observability

- **Existing.** Temporal documents `service_requests`, `service_latency`, `service_error_with_type`, `persistence_requests`, `persistence_errors`, `persistence_latency`, `no_poller_tasks`, `poll_timeouts`, and `workflow_failed` ([cluster metrics](https://docs.temporal.io/references/cluster-metrics)). The exposed suffixes are unverified.
- **Existing.** `postgres_exporter` listens on 9187, takes a password file through `--datasource.pass-file`, works with a role granted `pg_monitor`, and names database size `pg_database_size_bytes` ([README](https://github.com/prometheus-community/postgres_exporter/blob/v0.21.1/README.md), [pg_database.go](https://github.com/prometheus-community/postgres_exporter/blob/v0.21.1/collector/pg_database.go)).
- **Existing.** Grafana Alloy's `otelcol.receiver.prometheus` turns Prometheus metrics into OpenTelemetry metrics for other `otelcol` components ([Alloy reference](https://grafana.com/docs/alloy/latest/reference/components/otelcol/otelcol.receiver.prometheus/)).
- **Existing.** In ops' own repository, the host telemetry role runs Alloy on each host, reads the journal with relabel rules that set `service_name`, and sends logs over the OpenTelemetry Protocol with a queue on disk. The observability stack's scrapes each set sample, label, and body limits, and its validation refuses a scrape without them. ops read these on 2026-11-24.
- **Unverified.** The `up` series that a Prometheus scrape writes survives the conversion to the OpenTelemetry Protocol and arrives in Mimir under that name.
- **Existing.** The template logs to standard output at level `info`, set by `LOG_LEVEL`. The log format isn't in the template and stays unverified.
- **Existing.** The Temporal server exports OpenTelemetry spans when `OTEL_TRACES_EXPORTER=otlp` is set, and reads the standard `OTEL_EXPORTER_OTLP_*` variables ([tracing](https://github.com/temporalio/temporal/blob/v1.32.2/docs/development/tracing.md)).
- **Unverified.** Which server operations produce spans, and whether the UI exports any.
- **Unverified.** Postgres 16 writes JSON logs with `log_destination=jsonlog` while the logging collector is off. The Postgres documentation ties `jsonlog` to the collector.
- **Unverified.** Alloy in the project reads cAdvisor and file system metrics for the Temporal volume without extra privileges.
- **New.** The Alloy container, the job names, the dashboard, and the alert rules.

## Patterns this design copies

- **Existing.** The tailnet policy on ops main defines the Temporal frontend service on TCP 7233 and the UI service on TCP 443, grants them to the operator and the operator's devices, and tests both. No tag yet hosts either service (read 2026-11-24).

These are in ops' own repository, read by ops on 2026-11-24.

- **Existing.** The CA app forwards raw TCP from a tailnet service to the CA container, so the CA itself handles Transport Layer Security, and Compose publishes no host port.
- **Existing.** The 1Password Connect app serves HTTPS from its tailnet sidecar and proxies to the API container over the Compose network, with no published port.
- **Existing.** An open ops pull request adds a `grafana-tls` nginx container to the observability stack: it listens on container port 8443, mounts a host certificate directory read-only, proxies to Grafana for `grafana.tbhb.cloud`, and passes a Compose health check that runs `nginx -t`. Its role's health gate waits for every expected container to report running and healthy (branch `agent/o11y-tls-pr1b`, read 2026-11-24).
- **Existing.** The tailnet policy defines a Grafana Tailscale Service, and the observability stack keeps its dashboards as JSON files in its Grafana configuration, among them the capacity dashboard.
- **Unverified.** How the Grafana service serves both its tailnet name and `grafana.tbhb.cloud` with one certificate. Step 5 copies whatever the merged pattern does.
- **Existing.** The shared Postgres design uses scram-sha-256 for its connections. **Unverified:** its Alloy container and PgBouncer come from this design's brief, and the draft read on 2026-11-24 doesn't show them yet.
- **Existing.** The CA's leaf template accepts client certificates only with a service identity URI, and its documentation says its policy blocks client issuance today. It allows six server names. Leaves last 72 hours by default, with a 168-hour cap.
- **Unverified.** The Temporal server reloads its certificates through `TEMPORAL_TLS_REFRESH_INTERVAL`, and the UI reloads its client certificate without a restart.
- **Existing.** The CA's backup runs from a systemd timer, encrypts and checksums each archive, keeps seven, reports status over the OpenTelemetry Protocol, alerts after 36 hours without success, and has a restore drill on a scratch machine.
- **Existing.** A volume role puts one ext4 file system on a new logical volume and never resizes, reformats, or deletes one.
- **Existing.** The event gateway's Temporal target dials with host, port, namespace, and certificate files, and keeps events for 7 days ([event gateway](../event-gateway/C-protocol.md#temporal-target)).
- **Existing.** The operator's store decisions of 2026-11-12 give each consumer its own bucket and access key, and plan replication to the backup host.
- **Existing.** The object store's own documentation, read by ops on 2026-11-24, says it runs one interim store on the storage host, serves the tailnet only, and sets up no replication yet. So the dump leaves the data host, and a second copy on the backup host waits for the store's replication work.
- **Unverified.** The data host can reach the object store today. D1 asks for the grant.
- **Existing.** The operator placed Temporal on the data host on 2026-11-13, because that host has the extra disk.
