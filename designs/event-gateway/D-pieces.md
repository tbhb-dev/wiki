---
type: Design Appendix
title: Event gateway pieces
description: Each build step of the event gateway in detail, the day-one repository, and the Temporal outline.
tags: [event-gateway, plan, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Appendix D. Pieces

Back to [the design](README.md).

Line counts are estimates of Go code, not counting tests. Each step is one pull request.

## Day one in the new repository

Before step 1, the gateway maintainer creates tbhb-dev/event-gateway:

- Public, Apache-2.0, with `LICENSE`, `LICENSES/`, and `REUSE.toml`.
- The tbhb-dev/rules baseline: Bazel with `bazelisk test //...` as the one gate, mise for bazelisk and uv, the shared repository checks workflow, and the `conclusion` aggregate check.
- Rulesets as awrel has them in `.github/rulesets/main.json`: linear history, signed commits, squash merges only, one approving review, and required checks with no bypass.
- `AGENTS.md` pointing at the model roster, `CLAUDE.md` importing it, the pull request template and draft schema, and the deny rules for raw `gh pr` commands.
- Go module `go.tbhb.dev/event-gateway` ([D1](decisions.md)), with a vanity import entry in the module host's repository.
- Renovate through the shared presets.

`tbhbdev alpha repo bootstrap` applies the standard settings. The repository needs its maintainer named before step 1. No session owns it yet.

## Steps

| Step | Size | Contents | Depends on |
| --- | --- | --- | --- |
| 1. Ingest | ~350 | `cmd/event-gateway serve`, `POST /webhooks/github`, HMAC check, envelope, embedded NATS server with no listener, the `EVENTS` stream config with `MaxAge` and the duplicate window, publish with message ID and body-hash check, `/healthz`, OTLP setup. Tests: a repeat inside the window, a 409 on a changed body, and expiry under a short `MaxAge`. A measurement of publish latency with `SyncAlways` | 0, D3 |
| 2. Deploy | Config | Compose file with the service, built on the host from a pinned commit, and a `cloudflared` sidecar, tunnel route for the webhook paths only, secrets as files, named volume, collector network, second GitHub webhook | 1, D2, D5, D6 |
| 3. Read API | ~200 | `GET /v1/events?after=N` through an ordered consumer, with the subscription filter, bearer tokens, `event-gateway events tail` | 1 |
| 4. Grafana | ~250 | `POST /webhooks/grafana` with awrel's checks, allowlist, and IDs. Second contact point in Grafana | 1 |
| 5. Session server | ~380 | `/connect`, `gateway.v1`, subscription validation ported from awrel's rules, one ordered consumer per socket | 3 |
| 6. Resume | ~200 | chunked `hello`, consumer start at `resume_after + 1` or T0, `caught_up`, `gap` from the stream's first sequence. Smaller than before, since one consumer carries replay and live events | 5 |
| 7. MCP client | ~390 | `event-gateway mcp`, awrel's tools, channel notifications, state file with `seq`, one-time import of awrel's session file | 6 |
| 8. Harness config | Config | Gateway MCP entry for a canary, then everyone | 7 |
| 9. Redelivery | ~220 | A loop that lists failed GitHub deliveries since the last good one and asks GitHub to redeliver them. Retention moved to step 1 as stream config | 2, D4 |
| 10. Retire awrel | Config | Remove awrel's webhook and contact point, stop its deploy, archive with a pointer | 8 plus one week |
| 11. Temporal | Several | See the outline below | Temporal deployed |

If the client's filter port pushes step 5 or 7 past 400 lines, split the filter package into its own step first. awrel's filter tests port as table tests, which keeps parity checkable.

### Why step 1 first

Step 1 depends on nothing but the repository. After step 2 deploys it, it keeps every webhook on disk, which awrel never did, and its metrics show webhook volume and failures per event type. Step 3 makes that record usable by any consumer, including tools' `loop`, before sessions move. Each later step adds a target or a source and leaves the earlier ones working.

### Step 2 checks

- GitHub's delivery log for the new webhook shows 202 responses.
- The `EVENTS` stream's message count grows, and `event_gateway_ingest_total` appears in the metrics store.
- A request to any path other than the webhook paths gets 404 at the tunnel.

### Step 8 rollout

The harness configuration owner adds the gateway entry under a new server name for one canary session, beside awrel's entry, after every session host runs awrel v0.6.0 or later. On first start the gateway client imports awrel's per-session subscription file, and its first hello carries the full set and `"import": true`, with no `resume_after`. The server then replays each source from and including that source's T0 within the retention window and goes live. A session with no usable import (no file, or a malformed, unknown-version or empty one) starts live-only, with a warning logged when a file was rejected. A returning session sends its saved `seq` as `resume_after`. T0 for a source is its earliest stored event in the stream, which first exists once that source's webhook is added, GitHub at step 2 and Grafana at step 4. After a few days of matching events, the owner moves every session within that window. A rollback means a session reconnects to awrel, which kept receiving and kept its own state file. Prompts that name awrel's tools need the new server name. The change lists them in its description.

## Temporal outline

Not detailed. Temporal and its database are deployed by ops, on owned hosts, before this starts.

- **Gateway:** a `temporal` target. Config maps a subscription filter to an action: start a workflow type, or signal a running one, with `SignalWithStart` covering both. The workflow ID comes from the event, for example `pr:<repo>#<number>`, following the action key idea in [appendix B](B-earlier-designs.md). The request ID is the event's `delivery`, so a redelivery is a no-op. Its durable consumer makes delivery at least once.
- **Intent sessions:** the operator's small intent sessions write issues on GitHub. An `issues` event reaches the gateway and starts the matching workflow. No new source is needed.
- **Workers:** a worker runs activities that call `tbhbdev alpha agent exec`, then `wait`, and records the result. tools changes the launcher to accept a named non-interactive caller.
- **Coordination sessions** stop once workflows cover their work.
