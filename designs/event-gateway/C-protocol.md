<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Appendix C. Protocol

Back to [the design](README.md).

Everything here is proposed. None of it exists yet.

## Envelope

```json
{
  "type": "event",
  "seq": 48213,
  "provider": "github",
  "event": "pull_request",
  "delivery": "6f1c0e5a-0000-0000-0000-000000000000",
  "repository": "tbhb-dev/event-gateway",
  "received_at": "2026-11-21T18:04:05Z",
  "payload": {}
}
```

The wire keeps awrel's field name `repository` for the scope, so awrel's client code and filters port unchanged. For alerts it holds `kind/id`, as awrel does. `seq` and `received_at` are new. Inside the gateway, the same struct goes to every target.

Each source owns its normalizing:

- **GitHub:** `event` from `X-GitHub-Event`, `delivery` from `X-GitHub-Delivery`, `repository` from `repository.full_name` or null, and `payload` as the raw body. A `ping` gets 202 and isn't stored.
- **Grafana:** one envelope per alert, with awrel's scope labels, allowlist, and delivery ID rule. The route stores the allowlisted payload, not the raw body.

## Storage schema

```sql
CREATE TABLE events (
  seq         INTEGER PRIMARY KEY AUTOINCREMENT,
  provider    TEXT NOT NULL,
  event       TEXT NOT NULL,
  delivery    TEXT NOT NULL,
  repository  TEXT,
  received_at TEXT NOT NULL,
  body_sha256 BLOB NOT NULL,
  payload     BLOB NOT NULL,
  UNIQUE (provider, delivery)
);
CREATE TABLE cursors (
  target TEXT PRIMARY KEY,
  seq    INTEGER NOT NULL
);
```

A repeated `(provider, delivery)` with the same body hash is a duplicate and gets 202. A repeated delivery with a different body gets 409 and a log line, as github-metrics treats it. Bodies over a cap, 5 MiB by default, keep the envelope and replace the payload with a truncation marker. The retention sweep deletes rows older than the window but never past the lowest pull-target cursor, and it logs a warning when a stale cursor holds rows back.

## Session protocol

The subprotocol is `gateway.v1`. Everything awrel's `awrel.v1` defines stays: message types, the 4096-byte client cap, filter fields and their rules, error codes, and the ping fast path.

New:

- **`hello`**, the first client message: `{"type":"hello","resume_after":48190,"subscriptions":[...]}`. Sending subscriptions in `hello` closes the window between connect and subscribe in which awrel drops events. `resume_after` is optional, and a new session leaves it out.
- **Replay:** the server sends every stored event after `resume_after` that matches the subscriptions, in `seq` order, then `{"type":"caught_up","seq":48213}`, then live events. Live events that arrive during replay queue behind it, so order holds.
- **`gap`:** if `resume_after` is older than the oldest stored row, the server sends `{"type":"gap","from":...,"to":...}` before replay. The MCP client turns it into a notice, as awrel's client does for an outage.
- **Subscriptions** live in server memory per socket and in the client's state file. The client sends them again on each connect.

Auth is a bearer token in the handshake. The server keeps a file of `name:sha256(token)` lines and compares hashes in constant time. A token's name appears in spans and logs. A changed file applies on the next connect.

## MCP client

`event-gateway mcp` is a stdio MCP server with awrel's four tools, under the same names and schemas, and the same channel notification shape. It adds:

- the last delivered `seq`, kept per session key beside the subscriptions, written after each notification;
- a delivery ID set for the last 1,000 events, to drop repeats after a replay;
- an exit when stdin closes, which addresses awrel's orphan processes after an `/mcp` reconnect. The fix is unverified until a test reproduces the orphan.

Environment variables mirror awrel's with an `EVENT_GATEWAY_` prefix.

## Target interface

```go
type Target interface {
	Name() string
	Deliver(ctx context.Context, e Event) error
}
```

The runner gives each pull target a goroutine that reads from its cursor in batches, calls `Deliver`, and advances the cursor in the same transaction as any target bookkeeping. A failure backs off up to 5 minutes and is counted in a metric. A target that wants only some events carries the same subscription filter type the session server uses, set in config.

The session server registers as a push target: the log calls it after each commit, and it never holds a cursor of its own, since each client holds one.
