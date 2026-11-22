---
type: Design Draft
title: Event gateway
description: A self-hosted gateway that logs GitHub and Grafana events and delivers them to agent sessions and later Temporal.
tags: [event-gateway, webhooks, mcp, design]
status: draft
owner: Gateway maintainer
updated: "2026-11-22"
decisions_open: 0
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Event gateway

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Gateway maintainer | 2026-11-22 | 0 open decisions in [decisions.md](decisions.md) |

## Summary

The event gateway is a new public project, tbhb-dev/event-gateway, which the operator decided on. It's a self-hosted front door for event-driven work. It receives GitHub webhooks and Grafana alerts, verifies them, commits each one to a durable NATS JetStream stream inside the gateway, and then hands it to pluggable targets: agent sessions through its own Model Context Protocol (MCP) server, and later Temporal.

It replaces awrel. awrel runs on a hosted edge platform, keeps no events, and drops every event a session misses while disconnected. The gateway runs on an owned host, commits each event to its stream before it acknowledges the event, and lets a session resume from its last sequence number.

The first piece is the ingest: one Go service that verifies GitHub webhooks and commits them to the stream. With a second webhook it runs beside awrel, and the migration below gives each session a defined replay boundary.

## Proposal

Build one Go service, `event-gateway`, and one MCP client binary from the same module. The service runs as a container on one owned host. A Cloudflare Tunnel sidecar carries public webhook traffic to it, which copies the shape of the github-metrics receiver ([appendix A](A-evidence.md#github-metrics-receiver)).

The service has three layers:

- **Sources** take inbound requests. GitHub comes first, then Grafana. Each source verifies a Hash-based Message Authentication Code (HMAC) over the raw body before it parses anything, then normalizes the request into one event envelope.
- **The stream** is one NATS JetStream stream, `EVENTS`, served by a NATS server embedded in the gateway process ([D12](decisions.md)), with file storage on a named volume. Each accepted event is published on `events.<provider>.<event>`, takes the stream sequence as its `seq`, and is on disk before the source answers 202. A repeated delivery ID inside the duplicate window collapses into the first message.
- **Targets** read the stream. Each target is a plug-in behind one small Go interface, and each pull target reads through its own durable JetStream consumer. Routing rules in config decide which events reach which target and what each does with them ([D14](decisions.md#d14-how-do-events-route-to-targets)). The first target is the session server. Temporal comes later. Other targets, such as a file sink, a chat notice, or a second Temporal cluster, use the same interface.

Sessions keep awrel's model. A session runs a stdio MCP client that holds one WebSocket to the gateway, subscribes per provider, repository, and event with awrel's filters plus number and label filters on issue and pull request events ([D13](decisions.md)), and receives matching events as channel notifications. The client exposes the same four tools under the same names. The gateway adds a sequence number to each event and a resume cursor to each connection, so a session that reconnects within the retention window gets every event it missed.

awrel's code doesn't change. It keeps serving until the last session has moved, and then it retires.

## How it works

### Event envelope

Every source produces the same envelope. It extends awrel's `event` message, so the session protocol stays familiar. [Appendix C](C-protocol.md) has the full fields.

| Field | Meaning |
| --- | --- |
| `seq` | The `EVENTS` stream sequence, assigned when the publish is stored |
| `provider` | `github` or `grafana`, set by the route and never read from the body. `internal` is reserved for events published from inside, which [D16](decisions.md#d16-may-workflows-agent-runs-and-sm-publish-events-into-the-gateway) defers |
| `event` | GitHub's event name, or `alert.firing` and `alert.resolved` |
| `delivery` | GitHub's delivery ID, or a deterministic hash for Grafana, as awrel does |
| `scope` | `owner/repo` for GitHub, `kind/id` for alerts. awrel calls this `repository` |
| `received_at` | Commit time |
| `payload` | Raw body for GitHub. Allowlisted fields for Grafana, as awrel does |

### Durability

| What | Durable or best-effort | How |
| --- | --- | --- |
| Verified GitHub webhook, after 202 | Durable | Published to the stream, with its acknowledgement back, before the 202. The server syncs every write to disk, which is unverified until step 1 measures it |
| GitHub webhook while the gateway is down | Recoverable | GitHub records the failed delivery. A later piece requests redelivery ([D4](decisions.md)). Until then, redelivery is manual |
| Grafana alert, after 202 | Durable | Same commit rule. Grafana's own retry behavior is unverified |
| Session delivery | Kept for replay within retention | At least once. The client resumes from its last `seq` and drops repeated `delivery` IDs |
| Temporal delivery | Durable | At least once from a durable pull consumer, passing a reference to the event rather than its payload. The workflow ID with its reuse policy, and the workflow's check of `delivery`, make a repeat harmless. An event that can't be delivered parks ([D15](decisions.md#d15-how-does-the-temporal-target-deliver-an-event-and-what-does-it-do-with-one-it-cant-deliver)) |
| Pings, echoes, and status notices | Best-effort | Never stored |
| Events older than retention | Gone | The stream's `MaxAge` deletes each whole message at 7 days, whatever any consumer's position ([D3](decisions.md)) |

### Targets

A target implements `Name()` and `Deliver(ctx, Event) error`. A pull target, such as Temporal, gets a goroutine that reads a durable JetStream pull consumer named after the target instance and filtered to the subjects its rules name. It acknowledges each event after `Deliver` succeeds and asks JetStream to redeliver it with backoff on a transient failure, without limit.

A permanent failure parks the event on a separate `PARKED` stream and the target moves on, so one poison event can't stall it. JetStream keeps each consumer's position, so the gateway keeps no cursor table. The session server opens one ordered consumer per socket, which serves both the resume read and the live events after it.

Adding a target type means adding one package and one config block. Adding a route to an existing type is config and a restart ([appendix C](C-protocol.md#routing-rules)).

### Sessions

The session protocol is awrel's `awrel.v1` protocol with two additions, served under the subprotocol `gateway.v1`. The client sends `resume_after` with its last `seq` and its subscriptions in a `hello` when it connects, split across frames when the set exceeds awrel's 4096-byte frame cap. The server replays matching events from the stream, sends a `caught_up` marker, and then goes live. awrel's message types, filter fields, size limits, and error codes stay the same ([appendix C](C-protocol.md#session-protocol)).

The MCP client stores its subscriptions and its last `seq` per session key in its state directory, as awrel's client stores subscriptions today. A Claude Code session still needs the development channels flag. That's a harness constraint and stays the same.

Each client gets its own bearer token rather than one shared token. The gateway keeps a file of token hashes and names, so the operator can revoke one client without touching the others. The server reloads the file at least every 30 seconds and closes the sockets of revoked tokens. The token name goes into telemetry, never the token.

### Runtime and ingress

- **Language:** Go, matching the awrel client and the github-metrics receiver. The image is one static binary with cgo off on a distroless base, run as a non-root user with a read-only root file system.
- **Queue and storage:** NATS JetStream, from the `nats-server` Go package, runs inside the gateway process ([D12](decisions.md)). File storage on one named volume, one stream, one replica, one host, no cluster. The server starts with `SyncAlways` on, so the file store syncs each write to disk before the publish acknowledgement returns. The default syncs every 2 minutes, which could lose the last events on power loss and break the durability the 202 promises. That the acknowledgement waits for the sync, and what it costs per publish, are unverified until step 1 measures them ([appendix A](A-evidence.md#nats-server-and-jetstream)).
- **No NATS port:** the embedded server starts with `DontListen`, and the gateway connects to it through the client's `InProcessServer` option, so the server opens no client listener and no NATS port is exposed in the container or on the host. The monitoring, cluster, and leaf node ports stay unset.
- **Deploy:** Docker Compose, deployed by ops, with the image built on the host from a pinned commit. Secrets arrive as files from 1Password, as the github-metrics receiver reads them.
- **Ingress:** a Cloudflare Tunnel sidecar routes only the webhook paths to the service. The session endpoint stays on the private network ([D2](decisions.md)).
- **Telemetry:** OpenTelemetry traces, metrics, and logs over the OpenTelemetry Protocol (OTLP) to the loopback-only local collector, which forwards logs to Loki. The container joins the collector's network, as the github-metrics receiver does. The code starts with a small internal telemetry package and moves to tbhb-dev/telemetry once that library exists.

### Migration off awrel

1. **Run alongside.** Add a second GitHub webhook that points at the gateway, and a second Grafana contact point. awrel keeps its webhook and is unchanged, so both systems receive the same events. T0 for each source is the gateway's earliest stored event for that source, the first message on `events.<provider>.>` in the stream. A source's first event arrives once its webhook is added, which is GitHub at step 2 of the build and Grafana at step 4 ([D-pieces](D-pieces.md)). After 7 days, `MaxAge` moves T0 forward with the retention window. Only the gateway uses T0, to pick an imported session's replay start.
2. **Move one session.** The harness configuration owner adds the gateway's MCP entry for one canary session. awrel keeps each session's subscriptions in a file at `$XDG_STATE_HOME/awrel/sessions/<sha256 of the session key>.json` (under the default XDG state directory when the variable is unset), JSON with `version`, `endpoint`, and `subscriptions` (`internal/persist/persist.go` in the awrel repository). awrel's own client reloads that file on restart, but only when `endpoint` matches, and the gateway client starts with no state file. So the swap converts it: on first start, if its own state file for the session is missing and awrel's exists, the gateway client imports the `subscriptions` list once, ignoring `endpoint`, and writes its own state file. Its first hello then carries the full subscription set and `"import": true`. Every session host must run awrel v0.6.0 or later before the swap, since awrel's state file format came with v0.6.0 (awrel PR #62, `internal/persist`). The import is skipped, and the session starts live-only with a logged warning, when the file is malformed, has an unknown `version`, or has an empty `subscriptions` list. Once the gateway's own state file exists, awrel's file is never read again, so later awrel edits after a rollback don't flow back. Compare its events with awrel's for a few days.
3. **Move all sessions.** Swap the awrel entry for the gateway entry everywhere, one session at a time, each with the same reconnect. Rollback means a session reconnects to awrel, which kept receiving throughout and kept its own state file.
4. **Soak.** Wait one week with no session on awrel and match the gateway's stream counts against GitHub's delivery log.
5. **Cut the webhooks over.** Remove awrel's webhook and contact point only after every session has moved. Rollback is to add them back, since awrel's code and deploy are unchanged.
6. **Retire awrel.** Stop its deploy, mark the repository archived with a pointer to the gateway, and revoke its secrets.

The server, not the client, picks the replay start for an imported session. The client's first hello carries `"import": true` and no `resume_after`. For each subscribed source, the gateway replays from and including T0, that source's earliest stored event within retention, then goes live. The client never needs to know T0, so there is no off-by-one and no channel for T0. No event received after T0 is lost for a session that moves within the retention window. Events before T0 were delivered by awrel as before. A session with no import and no `resume_after`, brand new or with nothing to import, gets no replay. It is live-only from its first hello. A returning session sends its last saved `seq` as `resume_after`, and replay is strictly after it.

The replay can repeat events awrel already delivered during the parallel run. That is expected under at-least-once delivery, and clients already drop repeated `delivery` IDs within their window. At first start the delivery ID set is empty, so duplicates are possible and acceptable.

## Risks and unknowns

| Risk | Effect | What handles it | Evidence |
| --- | --- | --- | --- |
| The one host or the tunnel goes down | GitHub records failed deliveries, and sessions see nothing new | Redelivery piece ([D4](decisions.md)), plus a Grafana alert on webhook gaps | Unverified for this service. github-metrics runs the same shape |
| Payloads at rest hold private repository content | A disk or backup leak exposes it | `MaxAge` deletes every message at 7 days, volume on the owned host only, no payload in logs ([D3](decisions.md)) | New |
| Channel notifications stay experimental in Claude Code | A harness change breaks live delivery | Keep awrel's notification shape so one fix covers both | awrel `README.md` (appendix A) |
| Container to loopback collector path | Telemetry goes nowhere | Join the collector network, as github-metrics does | github-metrics `deploy/compose.yaml` |
| cloudflared needs `runc` rather than a sandboxed runtime | The sidecar runs with weaker isolation | Same accepted debt as github-metrics, and the service itself stays sandboxed | github-metrics `deploy/compose.yaml` |
| The launcher refuses some non-interactive callers | A Temporal worker can't launch reviewer runs | tools piece in the Temporal phase | tools `docs/reference/alpha/agent-configuration.md` (appendix A) |
| Grafana retry and repeat behavior | An alert lost while the gateway is down | Grafana's repeat interval sends firing alerts again. Unverified | Unverified |
| A session moves later than the retention window after T0 | It can miss events between T0 and its connect | Move every session within the retention window | New |
| The embedded server's memory and disk use | The gateway process grows or the volume fills, and ingest fails | Set JetStream's memory and store limits in the server options, and alert on volume use. Unset limits default to 75% of system memory and 75% of the free disk. The duplicate window keeps every message ID of the last 7 days in memory | nats-server source (appendix A). Use under load is unverified |
| One stream on one host | Losing the volume loses every retained event, and a stream fault stops all ingest | Accepted for now ([D12](decisions.md)). GitHub redelivery recovers webhooks, not events already stored | New |
| A workflow writes to GitHub and a rule matches its own write | A workflow starts itself again and again | One-shot workflow IDs with `reject_duplicate` stop repeats on one head. A `sender` filter is debt | New |
| The Temporal server or its database is down | The Temporal target holds back and its pending count grows, while sessions keep working | Retry without limit, and alert on pending age. Events wait up to 7 days | New |
| A workflow author skips the `delivery` check | A repeated signal does the work twice | The worker code's helper, and one-shot IDs for work that mustn't repeat | New |
| A bad rule or template ships | Every matching event parks | Alert on the first park, and `parked retry` within 7 days | New |
| NATS server upgrades change JetStream behavior | A dependency bump changes storage, dedupe, or sync behavior | Pin the `nats-server` and `nats.go` versions, and run the ingest, dedupe, and replay tests on every bump | New |

## Known debt

These gaps are known and not yet designed. Each goes into the first build that exercises it.

| Debt | Where it bites | First step that answers it |
| --- | --- | --- |
| Exiting on stdin close may not end awrel's orphan processes after an `/mcp` reconnect | Orphan clients keep sockets and tokens open | 7, with a test that reproduces the orphan |
| The organization webhook's event list must include `check_run` and `check_suite`, and the action types the filters accept for them aren't listed | A subscription on those events never matches | 2 and 5 |
| Appendix A reads awrel at a commit after `v0.7.1`, while sessions run the client at `v0.6.0` or later | Behavior described in appendix A may not be what sessions run today | 5, by pinning the filter tests to the version sessions run |
| Appendix A's claims outside its NATS and Temporal sections don't carry proven, existing, new, or unverified labels | A reader can't weigh each claim | Next revision of this design |
| The NATS server runs inside the gateway process, which is temporary | The queue lives and dies with one process on one host, and no other service can share it | A later decision on clustering or a separate NATS server. The `Target` interface and the stream subjects let that swap happen without changing targets |
| Step 8's owner is still named the harness configuration owner. No durable role for harness MCP configuration was found among the public role names | A reader can't tell which role edits the MCP config | 8, once that role has a name |
| The `label` filter takes printable ASCII names, as awrel's `name` filter does | A label with other characters can't be followed | 5, if a repository uses one |
| Rules reload only on restart | A route change restarts the gateway, which drops session sockets until clients reconnect | A later step, once routes change often |
| The Temporal target has no update or update-with-start action | A workflow that needs a reply to the event can't get one from the gateway | 11b, if a workflow needs it |
| No `sender` filter on rules or subscriptions | Loops through GitHub writes rely on workflow IDs alone | 11a, if a loop shows up |
| Internal publish isn't built | Workflows chain only through GitHub writes | [D16](decisions.md#d16-may-workflows-agent-runs-and-sm-publish-events-into-the-gateway), a later build step |
| Workflow ID templates read a fixed field list | A route that needs another payload field needs code | 11a, per field |
| Routing on sm's Handler field | A Request routes only on labels | [D17](decisions.md#d17-may-the-gateway-hold-a-credential-that-reads-org-project-fields-so-rules-can-match-a-requests-handler) |

## Plan

Each step is one pull request of under 400 code lines, unless it says config or lies outside the gateway. [Appendix D](D-pieces.md) has the detail and the size estimates.

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 0 | Repository: public, Apache-2.0, rules baseline, CI, rulesets | `bazelisk test //...` passes on `main` and the rulesets match awrel's | Gateway maintainer |
| 1 | Ingest: GitHub route, HMAC, envelope, embedded JetStream stream with `MaxAge`, dedupe, telemetry | A signed test delivery returns 202 and appears in the stream once, even when sent twice | Gateway maintainer |
| 2 | Deploy: compose, tunnel route, secrets, second GitHub webhook | GitHub's delivery log shows 202s and the stream grows | Ops |
| 3 | Read API, one event by `seq`, and `events tail` command on the private network | `event-gateway events tail --after N` prints the missed events | Gateway maintainer |
| 4 | Grafana route, ported from awrel's semantics | A signed test alert lands with awrel's delivery ID | Gateway maintainer |
| 5 | Session server: subscriptions, filters, per-client tokens | awrel's filter test cases pass against the gateway, and so do the number and label cases | Gateway maintainer |
| 6 | Resume and replay over the session socket | A client that drops for 10 minutes receives every matching event at least once, in order | Gateway maintainer |
| 7 | MCP client with awrel's tools and channel notifications | A canary session gets the same events from both relays | Gateway maintainer |
| 8 | Harness MCP config: canary, then everyone | Every session lists the gateway's tools and none lists awrel's | Harness configuration owner |
| 9 | Redelivery of failed GitHub deliveries | A forced outage heals with no manual redelivery | Gateway maintainer, ops for the credential |
| 10 | Cut awrel's webhooks and retire it | awrel's webhook is gone and the gateway is the only receiver | Ops, awrel owner |
| 11a | Routing rules for every pull target | One event reaches two targets through two rules, set by config alone | Gateway maintainer |
| 11b | Temporal target | A routed event starts or signals a workflow, and a repeat changes nothing | Gateway maintainer, ops |
| 11c | Parking | A poison event parks and the next event delivers | Gateway maintainer |
| 11d | Worker, outside the gateway | A workflow reads its event by `seq` and launches a run | Worker owner, tools |

Recommended first piece: step 1. It's small, needs nothing else from the gateway, and once step 2 deploys it, it keeps a durable record of every webhook. awrel has never kept that record. Step 3 then gives any consumer a way to catch up on events, before the session server exists.

The Temporal piece adds routing rules, a target that maps them to start, signal, and `SignalWithStart` calls, parking, and a worker that calls `tbhbdev alpha agent exec`. Temporal itself, its database, and its deploy belong to ops. The launcher's handling of non-interactive callers belongs to tools. [Appendix D](D-pieces.md#temporal-outline) outlines it.

## Appendices

- [Decisions](decisions.md): 0 open, then the decided record.
- [A. Evidence](A-evidence.md): awrel today, the github-metrics receiver, the launcher, the `nats-server` embedding, the Temporal SDK and server.
- [B. Earlier designs](B-earlier-designs.md): what older designs offer, and the constraints this design drops.
- [C. Protocol](C-protocol.md): envelope, session protocol, read API, target interface, routing rules, Temporal target, parking, internal publish, stream configuration.
- [D. Pieces](D-pieces.md): each step in detail, day-one repository contents, Temporal outline.
