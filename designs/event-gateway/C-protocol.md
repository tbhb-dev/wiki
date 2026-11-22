---
type: Design Appendix
title: Event gateway protocol
description: The envelope, session protocol, MCP client, target interface, and storage schema of the event gateway.
tags: [event-gateway, protocol, design]
status: draft
parent: README.md
---

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

The log opens SQLite with `journal_mode=WAL` and `synchronous=FULL`, so a commit has reached the disk before the source answers 202. Write-ahead logging alone, at its usual `synchronous=NORMAL`, can lose the last commits on power loss, which would break the durability the 202 promises.

A repeated `(provider, delivery)` with the same body hash is a duplicate and gets 202. A repeated delivery with a different body gets 409 and a log line, as github-metrics treats it. Bodies over a cap, 5 MiB by default, keep the envelope and replace the payload with a truncation marker plus the fields subscription filters read (`action`, `conclusion`, `name`, and `workflow_path`, where the event has them), so a filtered subscription still matches a truncated event.

Retention has two limits:

- **Payload age cap.** The sweep replaces the `payload` of every row older than the retention window, 7 days by default ([D3](decisions.md)), with an expiry marker, whatever any cursor says. No raw payload outlives the window, even behind a stalled target.
- **Row retention.** The sweep deletes envelope rows older than the window, but never past the lowest pull-target cursor, so a slow target still sees each event, without its payload. A cursor that holds rows back logs a warning and raises a metric, and rows older than 30 days go regardless, which the target then sees as a `gap`.

## Session protocol

The subprotocol is `gateway.v1`. Everything awrel's `awrel.v1` defines stays: message types, the 4096-byte cap on each client frame, filter fields and their rules, error codes, and the ping fast path.

New:

- **`hello`**, the first client message, sent in one or more frames: `{"type":"hello","resume_after":48190,"subscriptions":[...],"more":true}`. Each frame stays under the 4096-byte cap and carries as many subscriptions as fit. Every frame but the last sets `"more":true`, and the last sets `"more":false` or leaves it out. Only the first frame carries `resume_after`, which is optional. An imported session sets it to the T0 marker ([migration](README.md#migration-off-awrel)), and a session with no prior state leaves it out. The server accepts up to 256 subscriptions across the frames and answers more with awrel's error for an invalid message. It sends nothing to the socket until the last frame arrives, and it buffers live events from the first frame, so subscriptions in `hello` close the window between connect and subscribe in which awrel drops events. A socket that sends no final frame within 10 seconds is closed.
- **Why chunked frames:** a full subscription set can exceed 4096 bytes, and the client sends it again on every reconnect. Chunking keeps awrel's per-frame cap, which bounds what an unauthenticated or misbehaving client makes the server parse, and keeps the server stateless between connections. A larger negotiated limit would raise that bound for every client. A server-side subscription-set ID would need state that survives gateway restarts and expires, and a client whose ID was lost would need the chunked path anyway.
- **Replay:** with `resume_after`, the server sends every stored event after it that matches the subscriptions, in `seq` order. Without `resume_after` there is no replay and the session is live-only from its first hello. The gateway keeps T0 per source in its own store, set when that source's webhook is added, and the migration sends it as `resume_after`. Replay from T0 can repeat events another system already delivered, which at-least-once delivery allows. After replay, or at once when there is none, the server sends `{"type":"caught_up","seq":48213}`, then live events. Live events that arrive during replay queue behind it, so order holds.
- **`gap`:** if `resume_after` is older than the oldest stored row, the server sends `{"type":"gap","from":...,"to":...}` before replay. The MCP client turns it into a notice, as awrel's client does for an outage.
- **Subscriptions** live in server memory per socket and in the client's state file. The client sends them again, in `hello` frames, on each connect.

Auth is a bearer token in the handshake. The server keeps a file of `name:sha256(token)` lines and compares hashes in constant time. A token's name appears in spans and logs. The server reloads the file when it changes and at least every 30 seconds, and after each reload it closes every open socket whose token hash is gone, with close code 4401. A revoked client loses access within 30 seconds, not at its next reconnect.

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
