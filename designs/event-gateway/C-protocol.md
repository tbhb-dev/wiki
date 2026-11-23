---
type: Design Appendix
title: Event gateway protocol
description: The envelope, stream configuration, session protocol, MCP client, and target interface of the event gateway.
tags: [event-gateway, protocol, design]
status: draft
parent: README.md
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:2220073242f73359
    at: "2026-11-22T23:11:10Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore jetstream missingkey -->

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
- **Internal**, reserved and not built ([D16](decisions.md#d16-may-workflows-agent-runs-and-sm-publish-events-into-the-gateway)): `provider` is `internal`, set by the route. The caller's token name is the source, and the caller chooses `event` and `delivery`. The envelope adds an optional `cause` object with the `seq` of the event that led to this one and a `depth` ([internal publish](#internal-publish)). Sessions can't subscribe to `internal` events in the first build, so the session protocol doesn't change.

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

Dedupe uses the JetStream message ID. The duplicate window is 7 days, the same as `MaxAge`, which is the largest window the server accepts. A repeated `Nats-Msg-Id` inside the window isn't stored, and the acknowledgement carries the original message's sequence and `duplicate: true`. The source then reads that message by sequence and compares its `Event-Body-SHA256` header with the new body's hash. The same hash gets 202. A different hash gets 409 and a log line, as github-metrics treats it. A repeat after the original message expired is outside the window: the stream stores it as a new event with a new `seq`, targets see it again under at-least-once delivery, and the client's delivery ID set or the workflow's own `delivery` check drops it where they still hold the ID. The server keeps every message ID of the window in memory, whose size at real volume is unverified.

Bodies over a cap, 5 MiB by default, keep the envelope and replace the payload with a truncation marker plus the fields subscription filters read (`action`, `conclusion`, `name`, `workflow_path`, the item's `number`, its label names with the changed label, and a pull request's head SHA, where the event has them), so a filtered subscription or a routing rule still matches a truncated event, and a workflow ID template can still read the head SHA.

Retention is one limit. `MaxAge` deletes each whole message, envelope and payload together, 7 days after it was stored ([D3](decisions.md)), whatever any consumer's position. It replaces the two limits of this design's first merged revision, a payload age cap at 7 days and envelope row retention up to 30 days. Envelopes no longer outlive payloads, and a slow target gets no envelope-only tail: a consumer behind the window sees a `gap`. This is a deliberate simplification. The stream enforces one limit, and the gateway has no sweep to write or test.

## Session protocol

The subprotocol is `gateway.v1`. Everything awrel's `awrel.v1` defines stays: message types, the 4096-byte cap on each client frame, filter fields and their rules, error codes, and the ping fast path.

New:

- **`hello`**, the first client message, sent in one or more frames: `{"type":"hello","resume_after":48190,"subscriptions":[...],"more":true}`, or `{"type":"hello","import":true,"subscriptions":[...]}` on the first start of an imported session. Each frame stays under the 4096-byte cap and carries as many subscriptions as fit. Every frame but the last sets `"more":true`, and the last sets `"more":false` or leaves it out. Only the first frame carries `resume_after` and `import`, both optional and never together. A returning session sends its last saved `seq` as `resume_after`, and replay is strictly after it. An imported session on its first start sends `"import": true` and no `resume_after` ([migration](README.md#migration-off-awrel)). A hello with neither is live-only. The server accepts up to 256 subscriptions across the frames and answers more with awrel's error for an invalid message. It sends nothing to the socket until the last frame arrives. It notes the stream's last sequence when the first frame arrives, and a live-only session starts strictly after it, so the stream itself buffers live events from the first frame and subscriptions in `hello` close the window between connect and subscribe in which awrel drops events. A socket that sends no final frame within 10 seconds is closed.
- **Why chunked frames:** a full subscription set can exceed 4096 bytes, and the client sends it again on every reconnect. Chunking keeps awrel's per-frame cap, which bounds what an unauthenticated or misbehaving client makes the server parse, and keeps the server stateless between connections. A larger negotiated limit would raise that bound for every client. A server-side subscription-set ID would need state that survives gateway restarts and expires, and a client whose ID was lost would need the chunked path anyway.
- **Replay:** each socket gets one ordered consumer on `EVENTS`, filtered to the subjects of its subscribed providers, and the server applies the rest of each subscription filter as awrel does. With `resume_after`, the consumer starts at `resume_after + 1` (`DeliverByStartSequencePolicy` with `OptStartSeq`), so the server sends every stored event strictly after `resume_after` that matches the subscriptions, in `seq` order. With `import`, the server picks the start itself: the consumer starts at the beginning of the stream (`DeliverAllPolicy`), so for each subscribed source replay begins at and includes T0, that source's earliest stored event within retention. The client never sends or learns T0. Without either there is no replay and the session is live-only. Replay from T0 can repeat events another system already delivered, which at-least-once delivery allows. When it opens the consumer, the server notes the stream's last sequence. It sends `{"type":"caught_up","seq":48213}` with that sequence once the consumer reports `NumPending == 0`, or at once when there is no backlog. It doesn't wait for an event at that sequence, because a filtered consumer may never receive one. Then it keeps reading the same consumer for live events. One consumer carries replay and live events, so order holds without a queue.
- **`gap`:** if `resume_after + 1` is below the stream's first sequence (`FirstSeq` in the stream state), `MaxAge` has already deleted events the client missed. The server sends `{"type":"gap","from":48191,"to":50000}` before replay, with `from` set to `resume_after + 1` and `to` to the first sequence minus one, and replay starts at the first sequence. The MCP client turns it into a notice, as awrel's client does for an outage.
- **Number and label filters** ([D13](decisions.md)): `number` and `label` are optional, each a list of 1 to 8 entries, valid only for GitHub `issues`, `issue_comment`, `pull_request`, and `pull_request_review`. Like awrel's filters they apply to one exact provider, event, and repository subscription, and all filters on it must match. `number` holds positive integers and matches `issue.number` on `issues` and `issue_comment`, and `pull_request.number` on the other two. `label` holds names that match exactly, including case and colons, each 1 to 128 printable ASCII characters as awrel's `name`. It matches when the name is in `issue.labels[].name` or `pull_request.labels[].name`, or, on a `labeled` or `unlabeled` action, equals `label.name`, even when the item's label list is empty. An event with neither source matches no `label` filter. A filter on any other event gets awrel's `bad_subscription` error, as awrel gives for its own filters. Neither filter is required, because they narrow issue and pull request events and awrel requires a filter only for check and workflow events.
- **Subscriptions** live in server memory per socket and in the client's state file. The client sends them again, in `hello` frames, on each connect.

Auth is a bearer token in the handshake. The server keeps a file of `name:sha256(token)` lines and compares hashes in constant time. A token's name appears in spans and logs. The server reloads the file when it changes and at least every 30 seconds, and after each reload it closes every open socket whose token hash is gone, with close code 4401. A revoked client loses access within 30 seconds, not at its next reconnect.

## MCP client

`event-gateway mcp` is a stdio MCP server with awrel's four tools under the same names, the same channel notification shape, and awrel's argument schemas extended with `number` and `label` on `subscribe` and `unsubscribe`, because awrel's schema refuses unknown properties. It adds:

- the cursor, the last `seq` seen, kept per session key beside the subscriptions, written after each notification and after `caught_up` (from its `seq`), so a disconnect after `caught_up` and before the first matching event keeps the cursor;
- a delivery ID set for the last 1,000 events, to drop repeats after a replay;
- an exit when stdin closes, which addresses awrel's orphan processes after an `/mcp` reconnect. The fix is unverified until a test reproduces the orphan.

Environment variables mirror awrel's with an `EVENT_GATEWAY_` prefix.

## Read API

The read API serves the private network only ([D2](decisions.md#d2-should-the-session-endpoint-stay-off-public-ingress)) and takes the same bearer tokens as the session server.

- `GET /v1/events?after=N` returns the stored events strictly after `N` that match an optional subscription filter, through an ordered consumer, with a `gap` marker when `N + 1` is below the stream's first sequence.
- `GET /v1/events/{seq}` returns one full envelope with its payload, read with `Stream.GetMsg`. It answers 404 for a sequence never stored and 410 for one below the stream's first sequence, which `MaxAge` deleted. A Temporal activity uses it to read the event behind a reference ([Temporal target](#temporal-target)).

## Target interface

```go
type Target interface {
	Name() string
	Deliver(ctx context.Context, e Event) error
}

// A Deliver error wrapping ErrPermanent parks the event. Any other error retries.
var ErrPermanent = errors.New("permanent")
```

The `Target` interface stays the gateway's own abstraction over a JetStream consumer, so a target never imports the NATS client. `Name()` returns the instance name from config, not the type, so two targets of one type, such as two Temporal clusters, each get their own consumer. The runner gives each pull target a durable pull consumer named after `Name()`, with explicit acknowledgement and one unacknowledged message at a time, so delivery stays in `seq` order.

The consumer's `FilterSubjects` hold the subjects of the providers and events the target's rules name, such as `events.github.pull_request`, so the target never fetches events no rule can match. Rules still check every fetched event, so the subject filter only saves work. The runner fetches the next event, calls `Deliver`, and acknowledges it on success.

A failure is one of two classes. A transient error answers with `NakWithDelay`, whose delay backs off up to 5 minutes, with no attempt limit, so an outage of the target's backend holds the target back until it recovers. The attempt count and the consumer's pending count are metrics, and a Grafana alert on pending age shows a stalled target. A permanent error, an error wrapping `ErrPermanent`, or a panic inside `Deliver` parks the event ([parked events](#parked-events)). The runner acknowledges a parked event and moves to the next one, so a poison event costs one park, not a stalled target.

A filtered consumer skips sequences that don't match, so a jump in sequence is normal and doesn't mean a gap. Instead, before each fetch the runner compares the last sequence it acknowledged with the stream's first sequence. When the first sequence is more than one past it, `MaxAge` deleted events the target hadn't read, some of which may not have matched. The runner logs that possible `gap` and counts it in a metric.

The session server is the one target the runner doesn't drive. It opens an ordered consumer per socket, which is ephemeral and keeps no position, since each client holds its own last `seq`.

## Routing rules

Routes are config ([D14](decisions.md#d14-how-do-events-route-to-targets)). Each target instance has a name, a type, a start position, and a list of rules. A rule has a name, a match, and an action that the target type defines. An example, with sm's label names as placeholders that sm owns:

```yaml
targets:
  - name: temporal-main
    type: temporal
    start: new            # new, all, or a time such as 2026-11-20T00:00:00Z
    connection: { host_port: "temporal:7233", namespace: agents }
    rules:
      - name: review-pr-head
        match: { provider: github, event: pull_request, repository: ["tbhb-dev/*"], action: [opened, synchronize, ready_for_review] }
        action:
          kind: signal_with_start   # start, signal, or signal_with_start
          task_queue: reviews
          workflow_type: Review
          workflow_id: "review:{{.repository}}#{{.number}}@{{.head_sha}}"
          id_reuse: reject_duplicate
          signal: event
      - name: role-request
        match: { provider: github, event: issues, repository: ["tbhb-dev/*"], action: [labeled], label: ["role:reviewer"] }
        action: { kind: start, task_queue: requests, workflow_type: Request, workflow_id: "request:{{.repository}}#{{.number}}", id_reuse: allow_duplicate }
```

**Match.** A rule's match uses the subscription filter fields: `provider`, `event`, `repository`, and the filters `action`, `conclusion`, `name`, `workflow_path`, `number`, and `label` ([D13](decisions.md#d13-how-do-subscriptions-follow-one-issue-pull-request-or-label)). It's checked by the same filter package and the same match function the session server uses, so a rule and a subscription with the same fields match the same events. Validation differs in two places. A rule may list several repositories and may end a repository with `/*` for every repository of an owner, where a session subscription names exactly one. And a rule may name several events, where a subscription names one. The per-event filter tables are the same.

**Fan-out.** One event can match rules in any number of targets, since each target reads its own consumer. Within one target, every matching rule produces one action, in rule order, and the target acknowledges the event only when every action succeeds. A retry repeats every action, which idempotency makes safe.

**Templates.** A workflow ID is a Go `text/template` over a fixed set of fields the gateway extracts from the event: `provider`, `event`, `delivery`, `seq`, `repository`, `action`, `number`, `label` (the changed label on `labeled` and `unlabeled`), `head_sha`, and `sender`. Templates run with `missingkey=error`, so a template that reads a field the event lacks is a permanent error, and the event parks rather than starting a workflow with a broken ID. The gateway parses every template at start and refuses to start on a parse error.

**Config only.** A new rule on an existing target type, using existing filter fields and template fields, is a config change and a gateway restart. A restart loses nothing, since every durable consumer keeps its position on the volume. A new filter field, a new template field, a new action kind, or a new target type needs code. Hot reload of rules is left for later.

**Start and backfill.** `start` sets the consumer's deliver policy when the gateway first creates it: `new` (`DeliverNewPolicy`, the default), `all` (`DeliverAllPolicy`, everything still in the stream), or a time (`DeliverByStartTimePolicy` with `OptStartTime`). JetStream refuses to change a consumer's deliver policy or start time after it exists, but accepts new `FilterSubjects` ([appendix A](A-evidence.md#nats-server-and-jetstream)). So a new target backfills by its `start`, and a rule added to an existing target sees only events after the target's current position. To backfill one new rule, add it under a new target instance with its own name and `start`, and move it to the main instance once it's caught up, with the same workflow ID template so the overlap dedupes. Backfill reaches back at most 7 days. Events older than `MaxAge` are gone, and a workflow that needs them reconciles from GitHub itself, as the reconcile sweep of [appendix B](B-earlier-designs.md#review-and-merge-queues) does.

## Temporal target

The `temporal` target type holds one Temporal client per instance, made with `client.Dial` from the instance's `connection` block: host and port, namespace, and optional TLS credentials read from files. A second cluster is a second instance with its own name and connection.

**Actions.** `start` calls `ExecuteWorkflow` with the rule's task queue, workflow type, and workflow ID. `signal` calls `SignalWorkflow` on the workflow ID with an empty run ID, and a missing workflow is a permanent error. `signal_with_start` calls `SignalWithStartWorkflow`, which signals a running workflow or starts it and signals it in one call. `id_reuse` sets `WorkflowIDReusePolicy`: `allow_duplicate`, `allow_duplicate_failed_only`, or `reject_duplicate`. A start sets `WorkflowIDConflictPolicy` to `UseExisting`, so a start whose workflow is already running counts as delivered. Update and update-with-start are left for later.

**Argument.** The workflow input and the signal argument are the same reference, never the payload:

```json
{
  "seq": 48213, "provider": "github", "event": "pull_request", "delivery": "6f1c0e5a-...",
  "repository": "tbhb-dev/event-gateway", "received_at": "2026-11-21T18:04:05Z",
  "fields": { "action": "synchronize", "number": 12, "head_sha": "...", "label": null, "sender": "..." },
  "rule": "review-pr-head", "target": "temporal-main"
}
```

It stays under 4 KiB, far below Temporal's 2 MiB default blob limit, whatever the body's size. A workflow that needs the body runs an activity that calls `GET /v1/events/{seq}` and returns only what the workflow needs, because an activity result is a history event under the same limit. A 410 from the read API means the event expired, and the workflow decides what that means. Temporal workers therefore run on the private network and hold a read token.

**Idempotency.** Delivery is at least once, so the same reference can reach Temporal twice. The design doesn't rely on Temporal's request ID: the Go SDK's `SignalWithStartWorkflow` sets a random one that a caller can't choose, so a redelivery never repeats an ID, and the server drops only a repeated one ([appendix A](A-evidence.md#temporal)). Instead:

- A start with a one-shot ID, such as `review:<repo>#<n>@<head>`, uses `reject_duplicate`. A repeat while it runs hits `UseExisting`, and a repeat after it closed fails with an already-started error when the start options set `WorkflowExecutionErrorWhenAlreadyStarted`, which the target counts as delivered. Without that flag, `ExecuteWorkflow` returns the existing run's handle and no error.
- A signal can arrive twice at a running workflow. Each workflow keeps the `delivery` values it has handled, carried across continue-as-new, and drops a repeat. This is a rule for workflow authors, and the worker code gives them one helper for it.
- A long-lived ID, such as `pr:<repo>#<n>`, with `allow_duplicate`, can start a second run when a repeat arrives after the first run closed. Rules that can't accept that use `reject_duplicate`.

**Error classes.** Transient, so retried: gRPC `Unavailable`, `DeadlineExceeded`, `ResourceExhausted`, `Aborted`, `Internal`, a connection error, and a context timeout. Delivered, so acknowledged: an already-started error under `reject_duplicate`. Permanent, so parked: `InvalidArgument`, `NotFound` on `signal`, `PermissionDenied`, `NamespaceNotFound`, a failed template, and any other error the target doesn't recognize. Which already-started error type the server returns for each policy is unverified until step 11b tests it against a dev server.

## Parked events

A second stream, `PARKED`, on subjects `parked.<target>`, holds one record for each parked event: the target and rule names, the event's `seq`, `provider`, `event`, and `delivery`, the error text, and the time. It holds no payload, so [D3](decisions.md#d3-may-the-gateway-keep-raw-event-payloads-at-rest-for-7-days)'s 7-day limit on payloads still holds while the record keeps 30 days (`MaxAge` 30 days, file storage, one replica). `EVENTS` doesn't change.

`event-gateway parked list` prints the records, and `event-gateway parked retry <target> <seq>` reads the event from `EVENTS` and calls that target's `Deliver` once, which works only while the event is within the 7-day window. Each park counts in a metric, and a Grafana alert fires on the first one.

## Internal publish

Not built now. [D16](decisions.md#d16-may-workflows-agent-runs-and-sm-publish-events-into-the-gateway) keeps it possible, and this section records the shape so a later build step needs no change to the envelope, subjects, or `Target` interface.

`POST /v1/publish` on the private listener, never on the tunnel. The caller sends a bearer token from a separate file of publish tokens, in the session token file's format with names limited to lowercase letters, digits, and `-`, so a session token can't publish and a publish token can't connect. The body is JSON of at most 64 KiB: `event`, `delivery`, an optional `repository`, an optional `cause_seq`, and `payload`. `event` is 1 to 64 characters of lowercase letters, digits, `_`, and `.`. The gateway sets `provider` to `internal` and stores the message on `events.internal.<token name>.<event>` with `Nats-Msg-Id` set to `internal:<token name>:<delivery>`, so a caller's retry dedupes as a webhook's does and one caller can't collide with another's IDs. It answers 202 after the publish acknowledgement, as a source does. The subjects fall under `events.>`, so the stream config doesn't change.

**Loop protection.** With `cause_seq`, the gateway reads that event and sets `cause` to `{"seq": cause_seq, "depth": parent depth + 1}`, where an event with no `cause` has depth 0. It refuses a publish whose depth would exceed 8 with 422 and a log line, and refuses a `cause_seq` it can't read with 422. A workflow passes the `seq` of the reference that started it, so a chain that loops back to its own start stops at depth 8.

A workflow that publishes with no `cause_seq` starts a new chain, so the worker code always sets it from the workflow's input. Repeated starts of the same workflow also collapse on the workflow ID.

Loops that pass through GitHub, where a workflow writes a comment that a rule matches, carry no `cause`. Rules guard those with filters, and a `sender` filter is left for later.
