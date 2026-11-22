<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Appendix A. Evidence

Back to [the design](README.md).

Each claim here was read from `main` of a public repository on 2026-11-21. Paths are relative to that repository.

## awrel today

The read was taken at tbhb-dev/awrel commit `2c2ff52`, which is the merge of the Grafana alerts route (awrel#121). The latest tag is `v0.7.1`. Sessions still run the client at `v0.5.0`.

### Where it runs

- awrel is a Cloudflare Worker with one SQLite-backed Durable Object named `Relay`, written in TypeScript: `src/index.ts`, `src/relay.ts`, `src/connect.ts`, `src/github.ts`, `src/grafana.ts`, `src/env.ts`.
- It runs on Cloudflare's hosted platform, not on an owned host. It has no container, no systemd unit, and no tunnel. `wrangler.jsonc` defines a staging and a production environment, and custom domains carry ingress.
- `.github/workflows/deploy.yml` deploys staging and then production on each push to `main` from GitHub-hosted runners. It tests the new version at 0% traffic, cuts over, and rolls back on failure.
- `src/index.ts` routes `POST /webhooks/github`, `POST /webhooks/grafana`, and `GET /connect`, and answers 404 for anything else.
- The Go client lives in the same repository under module `go.tbhb.dev/awrel` (`go.mod`), with `cmd/awrel` and packages under `internal/`: `protocol`, `relayclient`, `mcpserver`, `delivery`, `compact`, `persist`, `logfile`, `transport`, `buildmeta`.

### Protocol

From `docs/reference/relay-protocol.md`:

- The handshake is `GET /connect` with a bearer token and the subprotocol `awrel.v1`.
- Each message is one JSON text frame. Client messages are capped at 4096 bytes.
- Client messages are `subscribe`, `unsubscribe`, `list`, `echo`, and `ping`. Server messages are `subscriptions`, `error`, `echoed`, `pong`, and `event`.
- The event envelope is `{type, provider, event, delivery, repository, payload}`.
- A subscription names `provider`, and for GitHub both `event` and `repository`. Optional filters are `action`, `conclusion`, `name`, and `workflow_path`. All filters must match, a list matches any member, and strings compare exactly.
- `check_run`, `workflow_run`, and `workflow_job` need `name` or `conclusion`, and `check_suite` needs `conclusion`. `action` works only for events in an explicit table.
- Subscriptions live in each socket's attachment, with a 12,288-byte budget. `Relay.publish` in `src/relay.ts` fans out to every matching socket from one Durable Object instance.
- The client pings every 30 seconds with a 10-second timeout, and redials with jittered backoff from 1 to 60 seconds (`internal/relayclient/relayclient.go`). On reconnect it replays its subscriptions.

### MCP client

- Tools are `subscribe`, `unsubscribe`, `list_subscriptions`, and `check_delivery` (`internal/mcpserver/mcpserver.go`). It exposes no resources.
- Events reach the session as channel notifications through the experimental `claude/channel` capability. Claude Code needs its development channels flag.
- Payloads are compacted by default (`internal/compact`), and payloads over 64 KiB become a short note.
- Configuration comes from `AWREL_ENDPOINT`, `AWREL_CONNECT_TOKEN` or `AWREL_CONNECT_TOKEN_FILE`, `AWREL_SESSION_ID`, and `AWREL_PERSIST`.
- Subscriptions persist per session key under the XDG state directory, mode 0600, pruned after 14 days.

### Auth and secrets

- GitHub webhooks are checked with HMAC-SHA256 over the raw body in `X-Hub-Signature-256` (`src/github.ts`). A `ping` event is answered and not relayed.
- Grafana alerts are checked with a hex HMAC-SHA256 in `X-Grafana-Alerting-Signature`, in constant time (`src/grafana.ts`). Bodies over 1 MiB get 413, and `alerts` must hold 1 to 100 entries.
- Sessions share one static bearer token, compared in constant time (`src/connect.ts`). There's no per-client identity.
- Worker secret names are `GITHUB_WEBHOOK_SECRET`, `GRAFANA_WEBHOOK_SECRET`, and `CONNECT_TOKEN`. A missing secret makes the Worker answer 503.

### Grafana route

`src/grafana.ts` publishes one event per firing or resolved alert that carries a scope label. The scope label maps to `kind/id`, which fills `repository`. The payload is an allowlist of alert name, status, scope, kind, value, start and end times, fingerprint, and link. The delivery ID is `gf-` plus a hash of the body and the alert's index, so a retried body keeps its IDs.

### Storage and telemetry

- awrel stores no events. The Durable Object's SQLite migration exists, but subscriptions live only in socket attachments.
- awrel has no OpenTelemetry. The Worker logs to the platform's log service, and the client writes structured logs without payloads.

### Known debt

- Events sent while a session is disconnected are lost, with no replay (`README.md`). The client emits a notice with the outage window.
- One shared token for every session.
- One Durable Object instance.
- A Claude Code `/mcp` reconnect leaves orphan client processes and double connections (`README.md`).
- Compacted payloads can still carry third-party text.
- Each new event needs a table update before `action` filtering works on it.

The open issue list wasn't read, because the read had no GitHub credentials.

## github-metrics receiver

tbhb-dev/github-metrics already receives GitHub webhooks through a tunnel on an owned host. The gateway copies its shape.

- Go with `net/http`: `internal/receiver/server.go`, `cmd/github-metrics/main.go`.
- `internal/receiver/receiver.go` checks `X-Hub-Signature-256` by streaming the body through HMAC-SHA256 under `http.MaxBytesReader` and comparing with `hmac.Equal`. It validates the event and delivery headers, answers 401, 400, or 413 on failure, and answers 202 only after the store call succeeds.
- `internal/store/store.go` keeps a queue and a dedupe table in one bbolt file. A reused delivery ID with a different body is an error.
- `deploy/compose.yaml` and `deploy/up.sh` run it under Docker Compose with an image pinned by digest, as user 65532, read-only root, all capabilities dropped, and no published port.
- A digest-pinned `cloudflared` sidecar on the same network carries the tunnel. It runs under `runc` because it can't resolve names under a sandboxed runtime, which the compose file records as debt. The tunnel routes only `POST /webhook` to the receiver, per `docs/architecture/decisions/0002-may-the-webhook-path-face-the-public-internet-th.md`.
- `internal/telemetry/telemetry.go` exports OTLP metrics when `OTEL_EXPORTER_OTLP_ENDPOINT` is set. Logs go to stdout as JSON. The container joins the collector's external network.
- Secrets are files read through `*_FILE` variables.

## Launcher

From tbhb-dev/tools, `src/tbhbdev/alpha/agent/` and `docs/reference/alpha/agent-configuration.md`:

- `tbhbdev alpha agent exec` takes a host, repository, model, effort, profile, run name, and prompt file. It starts a supervised headless run on the host. `wait`, `status`, `log`, and `diff` follow it.
- A launch records the calling session's terminal pane title. A reviewer launch is refused when that name is missing, so a non-interactive caller such as a Temporal worker needs a change in tools first.
- tools also holds `loop`, a shadow runner with a SQLite store of receipts and action keys. It's a likely consumer of gateway events.

## Telemetry library

No tbhb-dev/telemetry repository exists yet. The gateway starts with its own small telemetry package, as github-metrics does.
