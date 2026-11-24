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
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/pg
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - type: bind
        source: ${TEMPORAL_VOLUME:?}/pgdata
        target: /var/lib/postgresql/data/pgdata
        bind: {create_host_path: false}
      - ./.secrets/pg:/run/secrets/pg:ro
      - ${POSTGRES_LOG_DIR:?}:/var/log/postgresql
    command: ["postgres", "-c", "max_connections=200", "-c", "shared_buffers=1GB",
              "-c", "logging_collector=on", "-c", "log_destination=jsonlog",
              "-c", "log_directory=/var/log/postgresql"]
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres", "-d", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 12

  schema:
    <<: *hardening
    image: temporalio/admin-tools:1.32.2  # pin by digest at build
    restart: "no"
    depends_on: {postgres: {condition: service_healthy}}
    volumes:
      - ./schema-job.sh:/etc/temporal/schema-job.sh:ro
    env_file: ./.secrets/temporal.env
    command: ["sh", "/etc/temporal/schema-job.sh"]

  temporal:
    <<: *hardening
    image: temporalio/server:1.32.2  # pin by digest at build
    mem_limit: 2g
    depends_on: {schema: {condition: service_completed_successfully}}
    volumes:
      - ./certs/server:/certs:ro
    env_file: ./.secrets/temporal.env
    environment:
      DB: postgres12
      POSTGRES_SEEDS: postgres
      NUM_HISTORY_SHARDS: "128"          # D3, fixed after first start
      PROMETHEUS_ENDPOINT: ":8000"  # project network only
      TEMPORAL_TLS_REQUIRE_CLIENT_AUTH: "true"
      TEMPORAL_TLS_SERVER_CERT: /certs/server.pem
      TEMPORAL_TLS_SERVER_KEY: /certs/server-key.pem
      TEMPORAL_TLS_SERVER_CA_CERT: /certs/ca.pem
      TEMPORAL_TLS_FRONTEND_CERT: /certs/server.pem
      TEMPORAL_TLS_FRONTEND_KEY: /certs/server-key.pem
      TEMPORAL_TLS_CLIENT1_CA_CERT: /certs/ca.pem
      TEMPORAL_TLS_CLIENT2_CA_CERT: /certs/ca.pem
      TEMPORAL_TLS_INTERNODE_SERVER_NAME: temporal
      TEMPORAL_TLS_FRONTEND_SERVER_NAME: temporal
      TEMPORAL_TLS_REFRESH_INTERVAL: 1m
      TEMPORAL_ALLOW_NO_AUTH: "true"     # D2
      OTEL_TRACES_EXPORTER: otlp
      OTEL_EXPORTER_OTLP_ENDPOINT: http://alloy:4317
    healthcheck:
      test: ["CMD", "nc", "-z", "-w", "2", "temporal", "7233"]
      interval: 10s
      timeout: 5s
      retries: 12

  ui:
    <<: *hardening
    image: temporalio/ui  # newest release at build, pin by digest
    volumes:
      - ./certs/ui-client:/certs:ro
    environment:
      TEMPORAL_ADDRESS: temporal:7233
      TEMPORAL_TLS_CA: /certs/ca.pem
      TEMPORAL_TLS_CERT: /certs/client.pem
      TEMPORAL_TLS_KEY: /certs/client-key.pem
      TEMPORAL_TLS_SERVER_NAME: temporal
      TEMPORAL_DISABLE_WRITE_ACTIONS: "true"

  ui-tls:
    <<: *hardening
    image: nginx  # the observability stack's pin
    volumes:
      - ./ui-tls.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs/ui-proxy:/etc/nginx/certs:ro
    healthcheck:
      test: ["CMD", "nginx", "-t"]

  postgres-exporter:
    <<: *hardening
    image: prometheuscommunity/postgres-exporter:v0.21.1  # pin by digest at build

  alloy:
    <<: *hardening
    image: grafana/alloy  # the observability stack's pin
    volumes:
      - ${POSTGRES_LOG_DIR:?}:/var/log/postgresql:ro

  ts:
    image: tailscale/tailscale  # the version the other apps pin
    environment:
      TS_SERVE_CONFIG: /config/serve.json
    volumes:
      - ./serve.json:/config/serve.json:ro
```

`TEMPORAL_VOLUME` is the dedicated volume's host mount point. ops creates its `pgdata` subdirectory and owns it as the Postgres user; the bind fails if that subdirectory is missing. The host writes `.secrets/pg` from the superuser's `op://` item, readable by that user, and `.secrets/temporal.env` from the application role's item with `POSTGRES_USER`, `POSTGRES_PWD`, and the schema tool's `SQL_PASSWORD`. `POSTGRES_LOG_DIR` is a small mounted log volume, writable by Postgres and readable by Alloy.

Step 5 writes `schema-job.sh`: on the first deploy the `init` job runs `temporal-sql-tool --plugin postgres12 --ep postgres -u temporal --db <database> create` and `setup-schema -v 0.0` for both databases. The schema job runs `temporal-sql-tool --plugin postgres12 --ep postgres -u temporal --db <database> update-schema -d /etc/temporal/schema/postgresql/v12/<schema>/versioned` for `temporal` (schema `temporal`) and `temporal_visibility` (schema `visibility`), then exits with the command's status. It uses the application role and password from the environment. The Postgres health check runs `pg_isready`, satisfying the schema job's `service_healthy` dependency.

The server keeps its default ports inside the project: 7233 to 7239 for gRPC and 6933 to 6939 for membership. Nothing outside the network sees them. The server's client-auth variable sets the frontend and internode blocks together. Its `server.pem` has both server and client usages and the Compose service name `temporal` as a subject alternative name; the template also uses it for internode and built-in worker client connections. The UI mounts its own client certificate and key. Step 2 includes the frontend service's name in the server certificate for remote clients.

## Sidecar services

| Service | Port on the tailnet | Forward | Transport Layer Security ends at |
| --- | --- | --- | --- |
| Frontend | TCP 7233 | Raw TCP to `temporal:7233` | The Temporal server, which checks the client certificate |
| Web UI | TCP 443 | Raw TCP to `ui-tls:8443` | The `ui-tls` proxy, with the `temporal.tbhb.cloud` certificate |

The `ui-tls` proxy follows the observability stack's `grafana-tls` container: nginx, a certificate directory mounted read-only, container port 8443, and a Compose health check that runs `nginx -t`.

Step 5 writes `ui-tls.conf` with this listener and `serve.json` with these service forwards:

```nginx
server {
    listen 8443 ssl;
    server_name temporal.tbhb.cloud;
    ssl_certificate /etc/nginx/certs/server.pem;
    ssl_certificate_key /etc/nginx/certs/server-key.pem;
    location / { proxy_pass http://ui:8080; }
}
```

```json
{
  "Services": {
    "svc:temporal-frontend": {"TCP": {"7233": {"TCPForward": "temporal:7233"}}},
    "svc:temporal-ui": {"TCP": {"443": {"TCPForward": "ui-tls:8443"}}}
  }
}
```

## Credentials

Git holds `op://` references and nothing else. The vault holds five items: the Postgres superuser, the `temporal` role, the exporter's role, the object store key, and the backup passphrase. The private companion names each reference. Postgres uses Salted Challenge Response Authentication Mechanism (SCRAM) authentication for every role. Certificates come from the internal CA, and each private key stays on the host that uses it.

## Certificate renewal

CA leaves last 72 hours by default. A systemd timer on the data host renews the server and UI client certificates every 24 hours, and the server reloads them through `TEMPORAL_TLS_REFRESH_INTERVAL`. The gateway and the workers renew their own. The `temporal.tbhb.cloud` certificate for the proxy renews the way Grafana's does.

## Observability

Alloy in the project collects everything and sends it over the OpenTelemetry Protocol (OTLP) to the observability host's receiver, with a queue on disk.

| Signal | Source | Destination |
| --- | --- | --- |
| Metrics | `temporal:8000/metrics`, `postgres-exporter:9187/metrics`, cAdvisor for the containers, and file system metrics for the Temporal volume | Mimir |
| Logs | Postgres jsonlog files from the mounted log volume, and the Temporal server's logs from Docker | Loki |
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

The deploy's health gate waits until every long-running container reports `running` and `healthy` in `docker compose ps`, as the observability role's health tasks do. The server's Compose check uses the image's Alpine `nc` for a TCP check of `temporal:7233`. The deploy separately runs `temporal operator cluster health` from the matching `temporalio/admin-tools:1.32.2` image on the Compose project network, with `--address temporal:7233 --tls --tls-ca-path /certs/ca.pem --tls-cert-path /certs/client.pem --tls-key-path /certs/client-key.pem --tls-server-name temporal` and the UI client certificate directory mounted read-only at `/certs`. The server image contains no Temporal command-line tool.

## Backups

1. A systemd timer runs `pg_dump -Fc` on each database as the application role and `pg_dumpall --roles-only` as the Postgres superuser at night, from a Postgres 16 container on the project network. Each role's password comes from its existing `op://` item; the application role cannot read `pg_authid` for the roles dump.
2. It encrypts the files with the backup passphrase and writes a checksum beside each one.
3. It uploads them to Temporal's bucket in the object store, on another host.
4. The bucket's expiry rules keep 7 daily and 4 weekly sets.
5. A status timer reports the last result over OTLP, and the backup alert reads it.

Before each upgrade, ops runs a `CHECKPOINT`, takes a capped snapshot of the Temporal volume, and removes it within 24 hours after the checks pass. A snapshot on the same disk never counts as a backup.

## Restore drill

On a scratch machine with Docker, the drill downloads the newest set, checks it, decrypts it, and loads the roles file with `psql` into a fresh Postgres 16 first, because `pg_dump` omits cluster-wide roles. It then reinstalls each role's current SCRAM verifier from the credential store with `ALTER ROLE ... PASSWORD`, replacing any stale verifier in the dump, and restores the databases with `pg_restore`. It starts the same server version against it with `NUM_HISTORY_SHARDS=128` and no tailnet access. It passes when `temporal workflow list --namespace workflows` returns the dumped workflows.
