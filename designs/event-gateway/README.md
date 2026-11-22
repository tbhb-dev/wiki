---
type: Design Draft
title: Event gateway
description: A self-hosted gateway that logs GitHub and Grafana events and delivers them to agent sessions and later Temporal.
tags: [event-gateway, webhooks, mcp, design]
status: draft
owner: Event gateway owner
updated: "2026-11-22"
decisions_open: 6
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Event gateway

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Event gateway owner (new role) | 2026-11-22 | 6 open decisions in [decisions.md](decisions.md) |

## Summary

The event gateway is a new public project, tbhb-dev/event-gateway, which the operator decided on. It's a self-hosted front door for event-driven work. It receives GitHub webhooks and Grafana alerts, verifies them, writes each one to a local event log, and then hands it to pluggable targets: agent sessions through its own Model Context Protocol (MCP) server, and later Temporal.

It replaces awrel. awrel runs on a hosted edge platform, keeps no events, and drops every event a session misses while disconnected. The gateway runs on an owned host, commits each event to its log before it acknowledges the event, and lets a session resume from its last sequence number.

The first piece is the ingest: one Go service that verifies GitHub webhooks and commits them to the log. With a second webhook it runs beside awrel, and the migration below gives each session a defined replay boundary.

## Proposal

Build one Go service, `event-gateway`, and one MCP client binary from the same module. The service runs as a container on one owned host. A Cloudflare Tunnel sidecar carries public webhook traffic to it, which copies the shape of the github-metrics receiver ([appendix A](A-evidence.md#github-metrics-receiver)).

The service has three layers:

- **Sources** take inbound requests. GitHub comes first, then Grafana. Each source verifies a Hash-based Message Authentication Code (HMAC) over the raw body before it parses anything, then normalizes the request into one event envelope.
- **The log** is a SQLite file on a named volume. Each accepted event gets a monotonic sequence number and is committed before the source answers 202. Duplicate delivery IDs collapse into one row.
- **Targets** read the log. Each target is a plug-in behind one small Go interface, and each keeps its own cursor in the log. The first target is the session server. Temporal comes later. Other targets, such as a file sink or a chat notice, use the same interface.

Sessions keep awrel's model. A session runs a stdio MCP client that holds one WebSocket to the gateway, subscribes per provider, repository, and event with awrel's filters, and receives matching events as channel notifications. The client exposes the same four tools under the same names. The gateway adds a sequence number to each event and a resume cursor to each connection, so a session that reconnects within the retention window gets every event it missed.

awrel's code doesn't change. It keeps serving until the last session has moved, and then it retires.

## How it works

### Event envelope

Every source produces the same envelope. It extends awrel's `event` message, so the session protocol stays familiar. [Appendix C](C-protocol.md) has the full fields.

| Field | Meaning |
| --- | --- |
| `seq` | Gateway-wide sequence number, assigned at commit |
| `provider` | `github` or `grafana`, set by the route and never read from the body |
| `event` | GitHub's event name, or `alert.firing` and `alert.resolved` |
| `delivery` | GitHub's delivery ID, or a deterministic hash for Grafana, as awrel does |
| `scope` | `owner/repo` for GitHub, `kind/id` for alerts. awrel calls this `repository` |
| `received_at` | Commit time |
| `payload` | Raw body for GitHub. Allowlisted fields for Grafana, as awrel does |

### Durability

| What | Durable or best-effort | How |
| --- | --- | --- |
| Verified GitHub webhook, after 202 | Durable | Committed to the log before the 202 |
| GitHub webhook while the gateway is down | Recoverable | GitHub records the failed delivery. A later piece requests redelivery ([D4](decisions.md)). Until then, redelivery is manual |
| Grafana alert, after 202 | Durable | Same commit rule. Grafana's own retry behavior is unverified |
| Session delivery | Kept for replay within retention | At least once. The client resumes from its last `seq` and drops repeated `delivery` IDs |
| Temporal delivery | Durable | At least once from a per-target cursor. The workflow ID and request ID make it idempotent |
| Pings, echoes, and status notices | Best-effort | Never stored |
| Events older than retention | Gone | A sweep removes every payload at 7 days by default, whatever any target's cursor says ([D3](decisions.md)) |

### Targets

A target implements `Name()` and `Deliver(ctx, Event) error`. A pull target, such as Temporal, gets a goroutine that walks the log from its cursor and advances the cursor after each success, with backoff on failure. The session server is a push target. It fans each committed event out to connected sockets whose subscriptions match, and it serves resume reads straight from the log. Adding a target means adding one package and one config block.

### Sessions

The session protocol is awrel's `awrel.v1` protocol with two additions, served under the subprotocol `gateway.v1`. The client sends `resume_after` with its last `seq` and its subscriptions in a `hello` when it connects, split across frames when the set exceeds awrel's 4096-byte frame cap. The server replays matching events from the log, sends a `caught_up` marker, and then goes live. awrel's message types, filter fields, size limits, and error codes stay the same ([appendix C](C-protocol.md#session-protocol)).

The MCP client stores its subscriptions and its last `seq` per session key in its state directory, as awrel's client stores subscriptions today. A Claude Code session still needs the development channels flag. That's a harness constraint and stays the same.

Each client gets its own bearer token rather than one shared token. The gateway keeps a file of token hashes and names, so the operator can revoke one client without touching the others. The server reloads the file at least every 30 seconds and closes the sockets of revoked tokens. The token name goes into telemetry, never the token.

### Runtime and ingress

- **Language:** Go, matching the awrel client and the github-metrics receiver. The image is one static binary with cgo off on a distroless base, run as a non-root user with a read-only root file system.
- **Storage:** SQLite through a pure-Go driver, in write-ahead logging mode with `synchronous=FULL`, on one named volume. Write-ahead logging alone doesn't make a commit durable before the 202. One writer, one host, no cluster.
- **Deploy:** Docker Compose, deployed by the ops owner's roles, with images pinned by digest. Secrets arrive as files from 1Password, as the github-metrics receiver reads them.
- **Ingress:** a Cloudflare Tunnel sidecar routes only the webhook paths to the service. The session endpoint stays on the private network ([D2](decisions.md)).
- **Telemetry:** OpenTelemetry traces, metrics, and logs over the OpenTelemetry Protocol (OTLP) to the loopback-only local collector, which forwards logs to Loki. The container joins the collector's network, as the github-metrics receiver does. The code starts with a small internal telemetry package and moves to tbhb-dev/telemetry once that library exists.

### Migration off awrel

1. **Run alongside.** Add a second GitHub webhook that points at the gateway, and a second Grafana contact point. awrel keeps its webhook and is unchanged, so both systems receive the same events. The gateway records T0 for each source in its own store: when that source's webhook was added, which is GitHub at step 2 of the build and Grafana at step 4 ([D-pieces](D-pieces.md)). Its earliest stored event for a source is that source's T0.
2. **Move one session.** The harness configuration owner adds the gateway's MCP entry for one canary session. awrel keeps each session's subscriptions in a file at `$XDG_STATE_HOME/awrel/sessions/<sha256 of the session key>.json` (default `~/.local/state/awrel/sessions/`), JSON with `version`, `endpoint`, and `subscriptions` (`internal/persist/persist.go` in the awrel repository). awrel's own client reloads that file on restart, but only when `endpoint` matches, and the gateway client starts with no state file. So the swap converts it: on first start, if its own state file for the session is missing and awrel's exists, the gateway client imports the `subscriptions` list once, ignoring `endpoint`, and writes its own state file. Its first hello then carries the full subscription set. Compare its events with awrel's for a few days.
3. **Move all sessions.** Swap the awrel entry for the gateway entry everywhere, one session at a time, each with the same reconnect. Rollback means a session reconnects to awrel, which kept receiving throughout.
4. **Soak.** Wait one week with no session on awrel and match gateway log counts against GitHub's delivery log.
5. **Cut the webhooks over.** Remove awrel's webhook and contact point only after every session has moved. Rollback is to add them back, since awrel's code and deploy are unchanged.
6. **Retire awrel.** Stop its deploy, mark the repository archived with a pointer to the gateway, and revoke its secrets.

An imported session's first hello sends `resume_after` set to the T0 marker the gateway holds for the sources it subscribes to. Since `resume_after` is one `seq`, the client sends the lowest marker, which is GitHub's. The gateway replays matching events from there within the retention window, and then goes live. So no event received after T0 is lost for a session that moves within the retention window. Events before T0 were delivered by awrel as before. A session with no prior state, brand new and with nothing to import, sends no `resume_after` and gets no replay. It is live-only from its first hello.

The replay can repeat events awrel already delivered during the parallel run. That is expected under at-least-once delivery, and clients already drop repeated `delivery` IDs within their window. At first start the delivery ID set is empty, so duplicates are possible and acceptable.

## Risks and unknowns

| Risk | Effect | What handles it | Evidence |
| --- | --- | --- | --- |
| The one host or the tunnel goes down | GitHub records failed deliveries, and sessions see nothing new | Redelivery piece ([D4](decisions.md)), plus a Grafana alert on webhook gaps | Unverified for this service. github-metrics runs the same shape |
| Payloads at rest hold private repository content | A disk or backup leak exposes it | Retention sweep, volume on the owned host only, no payload in logs ([D3](decisions.md)) | New |
| Channel notifications stay experimental in Claude Code | A harness change breaks live delivery | Keep awrel's notification shape so one fix covers both | awrel `README.md` (appendix A) |
| Container to loopback collector path | Telemetry goes nowhere | Join the collector network, as github-metrics does | github-metrics `deploy/compose.yaml` |
| cloudflared needs `runc` rather than a sandboxed runtime | The sidecar runs with weaker isolation | Same accepted debt as github-metrics, and the service itself stays sandboxed | github-metrics `deploy/compose.yaml` |
| The launcher refuses some non-interactive callers | A Temporal worker can't launch reviewer runs | tools piece in the Temporal phase | tools `docs/reference/alpha/agent-configuration.md` (appendix A) |
| Grafana retry and repeat behavior | An alert lost while the gateway is down | Grafana's repeat interval sends firing alerts again. Unverified | Unverified |
| A session moves later than the retention window after T0 | It can miss events between T0 and its connect | Move every session within the retention window | New |

## Known debt

These gaps are known and not yet designed. Each goes into the first build that exercises it.

| Debt | Where it bites | First step that answers it |
| --- | --- | --- |
| Exiting on stdin close may not end awrel's orphan processes after an `/mcp` reconnect | Orphan clients keep sockets and tokens open | 7, with a test that reproduces the orphan |
| The organization webhook's event list must include `check_run` and `check_suite`, and the action types the filters accept for them aren't listed | A subscription on those events never matches | 2 and 5 |
| Appendix A reads awrel at a commit after `v0.7.1`, while sessions run the client at `v0.5.0` | Behavior described in appendix A may not be what sessions run today | 5, by pinning the filter tests to the version sessions run |
| Appendix A's claims don't carry proven, existing, new, or unverified labels | A reader can't weigh each claim | Next revision of this design |

## Plan

Each step is one pull request of under 400 code lines, unless it says config. [Appendix D](D-pieces.md) has the detail and the size estimates.

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 0 | Repository: public, Apache-2.0, rules baseline, CI, rulesets | `bazelisk test //...` passes on `main` and the rulesets match awrel's | Event gateway owner |
| 1 | Ingest: GitHub route, HMAC, envelope, SQLite log, dedupe, telemetry | A signed test delivery returns 202 and appears in the log once, even when sent twice | Event gateway owner |
| 2 | Deploy: compose, tunnel route, secrets, second GitHub webhook | GitHub's delivery log shows 202s and the log grows | Ops |
| 3 | Read API and `events tail` command on the private network | `event-gateway events tail --after N` prints the missed events | Event gateway owner |
| 4 | Grafana route, ported from awrel's semantics | A signed test alert lands with awrel's delivery ID | Event gateway owner |
| 5 | Session server: subscriptions, filters, per-client tokens | awrel's filter test cases pass against the gateway | Event gateway owner |
| 6 | Resume and replay over the session socket | A client that drops for 10 minutes receives every matching event once | Event gateway owner |
| 7 | MCP client with awrel's tools and channel notifications | A canary session gets the same events from both relays | Event gateway owner |
| 8 | Harness MCP config: canary, then everyone | Every session lists the gateway's tools and none lists awrel's | Harness configuration owner |
| 9 | Retention sweep and redelivery of failed GitHub deliveries | A forced outage heals with no manual redelivery | Event gateway owner, ops for the credential |
| 10 | Cut awrel's webhooks and retire it | awrel's webhook is gone and the gateway is the only receiver | Ops, awrel owner |
| 11 | Temporal target, outlined only | A routed event starts or signals a workflow | Event gateway owner, ops, tools |

Recommended first piece: step 1. It's small, needs nothing else from the gateway, and once step 2 deploys it, it keeps a durable record of every webhook. awrel has never kept that record. Step 3 then gives any consumer a way to catch up on events, before the session server exists.

The Temporal piece adds one target that maps routing rules to `SignalWithStart` calls, plus a worker that calls `tbhbdev alpha agent exec`. Temporal itself, its database, and its deploy belong to ops. The launcher's handling of non-interactive callers belongs to tools. [Appendix D](D-pieces.md#temporal-outline) outlines it.

## Appendices

- [Decisions](decisions.md): 6 open, then the decided record.
- [A. Evidence](A-evidence.md): awrel today, the github-metrics receiver, the launcher.
- [B. Earlier designs](B-earlier-designs.md): what older designs offer, and the constraints this design drops.
- [C. Protocol](C-protocol.md): envelope, session protocol, target interface, storage schema.
- [D. Pieces](D-pieces.md): each step in detail, day-one repository contents, Temporal outline.
