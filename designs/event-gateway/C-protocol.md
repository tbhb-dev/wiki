---
type: Design Appendix
title: Event gateway protocol
description: The envelope, stream configuration, session protocol, MCP client, and target interface of the event gateway.
tags: [event-gateway, protocol, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore jetstream -->

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

## Stream configuration

```go
// The embedded server.
server.Options{
	JetStream:  true,
	StoreDir:   "/data/nats", // on the named volume
	DontListen: true,         // no client listener, no NATS port
	SyncAlways: true,         // sync each write before the publish acknowledgement
	MaxPayload: 6 << 20,      // 6 MiB, room for the 5 MiB body cap and the envelope
}

// The one stream, created by the gateway at start.
jetstream.StreamConfig{
	Name:       "EVENTS",
	Subjects:   []string{"events.>"},
	Storage:    jetstream.FileStorage,
	Replicas:   1,
	MaxAge:     7 * 24 * time.Hour,
	Duplicates: 7 * 24 * time.Hour,
}
```

The gateway starts the server, waits for `ReadyForConnections`, and connects with `nats.InProcessServer`. Field names come from the nats-server and nats.go sources ([appendix A](A-evidence.md#nats-server-and-jetstream)). The memory and store limits get values in step 1 config. Unset, they default to most of the host's memory and free disk.

Each event is one message on `events.<provider>.<event>`, such as `events.github.pull_request` or `events.grafana.alert.firing`. The message body is the envelope without `seq`, and a reader takes `seq` from the message's stream sequence. Two headers go with it: `Nats-Msg-Id` set to `<provider>:<delivery>`, and `Event-Body-SHA256` with the hash of the raw body.

`SyncAlways` is a server option rather than a stream field, and it covers every stream on the server. With it on, the file store syncs each message block write to disk. Without it, the server syncs every 2 minutes by default, and a power loss could drop events that already got a 202. A source answers 202 only after the publish acknowledgement returns. That the acknowledgement waits for the sync, and the latency it adds, are unverified until step 1 measures them.

Dedupe uses the JetStream message ID. The duplicate window is 7 days, the same as `MaxAge`, which is the largest window the server accepts. A repeated `Nats-Msg-Id` inside the window isn't stored, and the acknowledgement carries the original message's sequence and `duplicate: true`. The source then reads that message by sequence and compares its `Event-Body-SHA256` header with the new body's hash. The same hash gets 202. A different hash gets 409 and a log line, as github-metrics treats it. A repeat after the original message expired is outside the window: the stream stores it as a new event with a new `seq`, targets see it again under at-least-once delivery, and the client's delivery ID set or Temporal's request ID drops it where they still hold the ID. The server keeps every message ID of the window in memory, whose size at real volume is unverified.

Bodies over a cap, 5 MiB by default, keep the envelope and replace the payload with a truncation marker plus the fields subscription filters read (`action`, `conclusion`, `name`, and `workflow_path`, where the event has them), so a filtered subscription still matches a truncated event.

Retention is one limit. `MaxAge` deletes each whole message, envelope and payload together, 7 days after it was stored ([D3](decisions.md)), whatever any consumer's position. It replaces the two limits of this design's first merged revision, a payload age cap at 7 days and envelope row retention up to 30 days. Envelopes no longer outlive payloads, and a slow target gets no envelope-only tail: a consumer behind the window sees a `gap`. This is a deliberate simplification. The stream enforces one limit, and the gateway has no sweep to write or test.

## Session protocol

The subprotocol is `gateway.v1`. Everything awrel's `awrel.v1` defines stays: message types, the 4096-byte cap on each client frame, filter fields and their rules, error codes, and the ping fast path.

New:

- **`hello`**, the first client message, sent in one or more frames: `{"type":"hello","resume_after":48190,"subscriptions":[...],"more":true}`, or `{"type":"hello","import":true,"subscriptions":[...]}` on the first start of an imported session. Each frame stays under the 4096-byte cap and carries as many subscriptions as fit. Every frame but the last sets `"more":true`, and the last sets `"more":false` or leaves it out. Only the first frame carries `resume_after` and `import`, both optional and never together. A returning session sends its last saved `seq` as `resume_after`, and replay is strictly after it. An imported session on its first start sends `"import": true` and no `resume_after` ([migration](README.md#migration-off-awrel)). A hello with neither is live-only. The server accepts up to 256 subscriptions across the frames and answers more with awrel's error for an invalid message. It sends nothing to the socket until the last frame arrives. It notes the stream's last sequence when the first frame arrives, and a live-only session starts strictly after it, so the stream itself buffers live events from the first frame and subscriptions in `hello` close the window between connect and subscribe in which awrel drops events. A socket that sends no final frame within 10 seconds is closed.
- **Why chunked frames:** a full subscription set can exceed 4096 bytes, and the client sends it again on every reconnect. Chunking keeps awrel's per-frame cap, which bounds what an unauthenticated or misbehaving client makes the server parse, and keeps the server stateless between connections. A larger negotiated limit would raise that bound for every client. A server-side subscription-set ID would need state that survives gateway restarts and expires, and a client whose ID was lost would need the chunked path anyway.
- **Replay:** each socket gets one ordered consumer on `EVENTS`, filtered to the subjects of its subscribed providers, and the server applies the rest of each subscription filter as awrel does. With `resume_after`, the consumer starts at `resume_after + 1` (`DeliverByStartSequencePolicy` with `OptStartSeq`), so the server sends every stored event strictly after `resume_after` that matches the subscriptions, in `seq` order. With `import`, the server picks the start itself: the consumer starts at the beginning of the stream (`DeliverAllPolicy`), so for each subscribed source replay begins at and includes T0, that source's earliest stored event within retention. The client never sends or learns T0. Without either there is no replay and the session is live-only. Replay from T0 can repeat events another system already delivered, which at-least-once delivery allows. When it opens the consumer, the server notes the stream's last sequence. After it has sent every matching event up to that sequence, or at once when there is none, it sends `{"type":"caught_up","seq":48213}` with that sequence, then keeps reading the same consumer for live events. One consumer carries replay and live events, so order holds without a queue.
- **`gap`:** if `resume_after + 1` is below the stream's first sequence (`FirstSeq` in the stream state), `MaxAge` has already deleted events the client missed. The server sends `{"type":"gap","from":48191,"to":50000}` before replay, with `from` set to `resume_after + 1` and `to` to the first sequence minus one, and replay starts at the first sequence. The MCP client turns it into a notice, as awrel's client does for an outage.
- **Subscriptions** live in server memory per socket and in the client's state file. The client sends them again, in `hello` frames, on each connect.

Auth is a bearer token in the handshake. The server keeps a file of `name:sha256(token)` lines and compares hashes in constant time. A token's name appears in spans and logs. The server reloads the file when it changes and at least every 30 seconds, and after each reload it closes every open socket whose token hash is gone, with close code 4401. A revoked client loses access within 30 seconds, not at its next reconnect.

## MCP client

`event-gateway mcp` is a stdio MCP server with awrel's four tools, under the same names and schemas, and the same channel notification shape. It adds:

- the cursor, the last `seq` seen, kept per session key beside the subscriptions, written after each notification and after `caught_up` (from its `seq`), so a disconnect after `caught_up` and before the first matching event keeps the cursor;
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

The `Target` interface stays the gateway's own abstraction over a JetStream consumer, so a target never imports the NATS client. The runner gives each pull target a durable pull consumer named after `Name()`, with explicit acknowledgement and one unacknowledged message at a time, so delivery stays in `seq` order. It fetches the next event, calls `Deliver`, and acknowledges it on success. A failure answers with `NakWithDelay`, whose delay backs off up to 5 minutes and is counted in a metric. JetStream keeps each consumer's position on the volume, so there is no cursor table. A pull consumer reads the whole stream, so when the next event's sequence is more than one past the last acknowledged one, `MaxAge` deleted the events between, and the runner logs that `gap` and counts it in a metric. A target that wants only some events carries the same subscription filter type the session server uses, set in config.

The session server is the one target the runner doesn't drive. It opens an ordered consumer per socket, which is ephemeral and keeps no position, since each client holds its own last `seq`.
