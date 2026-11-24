---
type: Design Appendix
title: Self-hosted Temporal server deployment
description: The Compose sketch, ports, credentials, observability, and backups of the Temporal server.
tags: [temporal, deployment, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Self-hosted Temporal server deployment

Back to [the design](README.md).

This appendix sketches what plan step 5 builds. The build run writes the real files, and its tests check them. The private companion holds the host, volume, path, tag, and item names.

## Compose sketch

The sketch follows the other apps' pattern: digest pins, `cap_drop: [ALL]`, `no-new-privileges`, a memory and process limit on each container, no `ports:`, and one network.

```yaml
name: temporal

x-hardening: &hardening
  cap_drop: [ALL]
  security_opt: [no-new-privileges:true]

services:
  postgres:
    <<: *hardening
    image: postgres:16  # pin by digest at build
    user: "999:999"  # a root start needs capabilities that cap_drop removes; ops pre-owns the data directory
    mem_limit: 4g
    command: ["postgres", "-c", "max_connections=200", "-c", "shared_buffers=1GB",
              "-c", "logging_collector=off", "-c", "log_destination=jsonlog"]
    # data directory one level below the volume's mount point

  schema:
    <<: *hardening
    image: temporalio/admin-tools:1.32.2  # pin by digest at build
    restart: "no"
    depends_on: {postgres: {condition: service_healthy}}

  temporal:
    <<: *hardening
    image: temporalio/server:1.32.2  # pin by digest at build
    mem_limit: 2g
    depends_on: {schema: {condition: service_completed_successfully}}
    environment:
      DB: postgres12
      POSTGRES_SEEDS: postgres
      NUM_HISTORY_SHARDS: "128"          # D3, fixed after first start
      PROMETHEUS_ENDPOINT: 0.0.0.0:8000  # project network only
      TEMPORAL_TLS_REQUIRE_CLIENT_AUTH: "true"
      TEMPORAL_ALLOW_NO_AUTH: "true"     # D2
      OTEL_TRACES_EXPORTER: otlp
      OTEL_EXPORTER_OTLP_ENDPOINT: http://alloy:4317

  ui:
    <<: *hardening
    image: temporalio/ui  # newest release at build, pin by digest
    environment:
      TEMPORAL_ADDRESS: temporal:7233
      TEMPORAL_DISABLE_WRITE_ACTIONS: "true"

  ui-tls:
    <<: *hardening
    image: nginx  # the observability stack's pin
    # listens on 8443 with the host-provided certificate, proxies to ui:8080

  postgres-exporter:
    <<: *hardening
    image: prometheuscommunity/postgres-exporter:v0.21.1  # pin by digest at build

  alloy:
    <<: *hardening
    image: grafana/alloy  # the observability stack's pin

  ts:
    image: tailscale/tailscale  # the version the other apps pin
```

The server keeps its default ports inside the project: 7233 to 7239 for gRPC and 6933 to 6939 for membership. Nothing outside the network sees them. The server's client-auth variable sets the frontend and internode blocks together, so the server needs its own client certificate. If the template can't express that, step 5 mounts a full config file, checked with `temporal-server render-config`.

## Sidecar services

| Service | Port on the tailnet | Forward | Transport Layer Security ends at |
| --- | --- | --- | --- |
| Frontend | TCP 7233 | Raw TCP to `temporal:7233` | The Temporal server, which checks the client certificate |
| Web UI | TCP 443 | Raw TCP to `ui-tls:8443` | The `ui-tls` proxy, with the `temporal.tbhb.cloud` certificate |

The `ui-tls` proxy follows the observability stack's `grafana-tls` container: nginx, a certificate directory mounted read-only, container port 8443, and a Compose health check that runs `nginx -t`.

## Credentials

Git holds `op://` references and nothing else. The vault holds five items: the Postgres superuser, the `temporal` role, the exporter's role, the object store key, and the backup passphrase. The private companion names each reference. Postgres uses Salted Challenge Response Authentication Mechanism (SCRAM) authentication for every role. Certificates come from the internal CA, and each private key stays on the host that uses it.

## Certificate renewal

CA leaves last 72 hours by default. A systemd timer on the data host renews the server and UI client certificates every 24 hours, and the server reloads them through `TEMPORAL_TLS_REFRESH_INTERVAL`. The gateway and the workers renew their own. The `temporal.tbhb.cloud` certificate for the proxy renews the way Grafana's does.

## Observability

Alloy in the project collects everything and sends it over the OpenTelemetry Protocol (OTLP) to the observability host's receiver, with a queue on disk.

| Signal | Source | Destination |
| --- | --- | --- |
| Metrics | `temporal:8000/metrics`, `postgres-exporter:9187/metrics`, cAdvisor for the containers, and file system metrics for the Temporal volume | Mimir |
| Logs | JSON from Postgres and the Temporal server, read from Docker | Loki |
| Traces | The server's OpenTelemetry spans | Tempo |

Some series to watch: `service_requests`, `service_latency`, `persistence_latency`, `persistence_errors`, `no_poller_tasks`, task queue backlog, `pg_database_size_bytes`, and `pg_stat_activity_count` against `max_connections`. Temporal documents names without the suffixes its Prometheus reporter adds, so step 5 reads the exposed names before it writes the alert rules.

One dashboard, `Temporal`, shows the server, Postgres, and container panels. ops builds it live in Grafana, then backports its JSON into the observability stack's dashboard files.

| Alert | Condition | For |
| --- | --- | --- |
| Instance down | `up` is 0 for the server or the exporter | 5 minutes |
| Volume fill | The Temporal volume or the volume group's free space below 15% | 30 minutes |
| Backup failure | No successful backup for 36 hours, or a failed last run | Immediately |
| Connection saturation | Postgres connections over 80% of `max_connections` | 10 minutes |
| Persistence latency | The 95th percentile of `persistence_latency` over 1 second | 10 minutes |
| Task backlog | Task queue backlog growing with no pollers | 15 minutes |

The deploy's health gate waits until every long-running container reports `running` and `healthy` in `docker compose ps`, as the observability role's health tasks do. The server's check runs `temporal operator cluster health` from inside the project.

## Backups

1. A systemd timer runs `pg_dump -Fc` on both databases and `pg_dumpall --globals-only` at night, from a Postgres 16 container on the project network.
2. It encrypts the files with the backup passphrase and writes a checksum beside each one.
3. It uploads them to Temporal's bucket in the object store, on another host.
4. The bucket's expiry rules keep 7 daily and 4 weekly sets.
5. A status timer reports the last result over OTLP, and the backup alert reads it.

Before each upgrade, ops runs a `CHECKPOINT`, takes a capped snapshot of the Temporal volume, and removes it within 24 hours after the checks pass. A snapshot on the same disk never counts as a backup.

## Restore drill

On a scratch machine with Docker, the drill downloads the newest set, checks it, decrypts it, and loads the roles file with `psql` into a fresh Postgres 16 first, because `pg_dump` omits cluster-wide roles, and then restores the databases with `pg_restore`. It starts the same server version against it with no tailnet access. It passes when `temporal workflow list --namespace workflows` returns the dumped workflows.
