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

Line counts are estimates of Go code, not counting tests. No size here is measured, since no step is built. Each step is one pull request.

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
| 1. Ingest | ~350 | `cmd/event-gateway serve`, `POST /webhooks/github`, HMAC check, envelope, embedded NATS server with no listener, the `EVENTS` stream config with `MaxAge` and the duplicate window, publish with message ID and body-hash check, `/healthz`, OTLP setup. Tests: a repeat inside the window, a 409 on a changed body, and expiry under a short `MaxAge`. A measurement of publish latency with `SyncAlways`, and the memory the 7-day duplicate map takes at the expected event volume | 0, D3, D12 |
| 2. Deploy | Config | Compose file with the service, built on the host from a pinned commit, and a `cloudflared` sidecar, tunnel route for the webhook paths only, secrets as files, named volume, collector network, second GitHub webhook | 1, D2, D5, D6 |
| 3. Read API | ~240 | `GET /v1/events?after=N` through an ordered consumer, with the subscription filter, bearer tokens, `event-gateway events tail`, and `GET /v1/events/{seq}` for one full event, about 40 of the lines | 1 |
| 4. Grafana | ~250 | `POST /webhooks/grafana` with awrel's checks, allowlist, and IDs. Second contact point in Grafana | 1 |
| 5. Session server | ~380 | `/connect`, `gateway.v1`, subscription validation ported from awrel's rules plus the `number` and `label` filters ([D13](decisions.md)), one ordered consumer per socket | 3 |
| 6. Resume | ~200 | chunked `hello`, consumer start at `resume_after + 1` or T0, `caught_up`, `gap` from the stream's first sequence. Smaller than before, since one consumer carries replay and live events | 5 |
| 7. MCP client | ~390 | `event-gateway mcp`, awrel's tools, channel notifications, state file with `seq`, one-time import of awrel's session file | 6 |
| 8. Harness config | Config | Gateway MCP entry for a canary, then everyone | 7 |
| 9. Redelivery | ~220 | A loop that lists failed GitHub deliveries since the last good one and asks GitHub to redeliver them. Retention moved to step 1 as stream config | 2, D4 |
| 10. Retire awrel | Config | Remove awrel's webhook and contact point, stop its deploy, archive with a pointer | 8 plus one week |
| 11a. Rule engine | ~250 | Target instances and rules from config ([D14](decisions.md#d14-how-do-events-route-to-targets)), rule validation on the shared filter package, workflow ID templates and their field extractor, `FilterSubjects` and `start` on each durable consumer, and the possible-gap check. Tests: fan-out to two targets, a rule on a truncated event, a template with a missing field, an owner wildcard, consumer progress after `FilterSubjects` widens, and consumer progress after `MaxAge` deletes unread messages | 5 |
| 11b. Temporal target | ~280 | The `temporal` type: connection from config, `start`, `signal`, and `signal_with_start`, the reference argument, reuse and conflict policies, and error classes ([D15](decisions.md#d15-how-does-the-temporal-target-deliver-an-event-and-what-does-it-do-with-one-it-cant-deliver)). Tests against Temporal's dev server for each action, a repeat under each reuse policy with `WorkflowExecutionErrorWhenAlreadyStarted` set and unset, the error a closed workflow returns under `REJECT_DUPLICATE`, and an outage | 3, 11a, Temporal deployed |
| 11c. Parking | ~150 | `ErrPermanent`, panic recovery, the `PARKED` stream, `event-gateway parked list` and `retry`, the park metric and alert | 11a |
| 11d. Worker | Outside the gateway | The worker that runs activities, reads events by `seq`, and calls `tbhbdev alpha agent exec`, plus the helper that drops repeated `delivery` values. Its repository and size are for the worker's owner | 11b, the tools launcher change |

If the client's filter port pushes step 5 or 7 past 400 lines, split the filter package into its own step first. awrel's filter tests port as table tests, which keeps parity checkable. The `number` and `label` filters add about 60 lines, an estimate no build has measured, and the filter package is the likeliest split. Their cases cover a pull request with no labels, a `labeled` action, an `unlabeled` action for a followed label, a comment on a pull request, a filter on an unsupported event, a list over 8 entries, and duplicate entries.

### Why step 1 first

Step 1 depends on nothing but the repository. After step 2 deploys it, it keeps every webhook on disk, which awrel never did, and its metrics show webhook volume and failures per event type. Step 3 makes that record usable by any consumer, including tools' `loop`, before sessions move. Each later step adds a target or a source and leaves the earlier ones working.

### Step 2 checks

- GitHub's delivery log for the new webhook shows 202 responses.
- The `EVENTS` stream's message count grows, and `event_gateway_ingest_total` appears in the metrics store.
- A request to any path other than the webhook paths gets 404 at the tunnel.

### Step 8 rollout

The harness configuration owner adds the gateway entry under a new server name for one canary session, beside awrel's entry, after every session host runs awrel v0.6.0 or later. On first start the gateway client imports awrel's per-session subscription file, and its first hello carries the full set and `"import": true`, with no `resume_after`. The server then replays each source from and including that source's T0 within the retention window and goes live. A session with no usable import (no file, or a malformed, unknown-version or empty one) starts live-only, with a warning logged when a file was rejected. A returning session sends its saved `seq` as `resume_after`. T0 for a source is its earliest stored event in the stream, which first exists once that source's webhook is added, GitHub at step 2 and Grafana at step 4. After a few days of matching events, the owner moves every session within that window. A rollback means a session reconnects to awrel, which kept receiving and kept its own state file. Prompts that name awrel's tools need the new server name. The change lists them in its description.

## Temporal outline

Temporal and its database are deployed by ops, on owned hosts, before step 11b starts. [Appendix C](C-protocol.md#routing-rules) specifies the rules, the Temporal target, parking, and internal publish.

- **Gateway:** routing rules in config map events to target instances ([D14](decisions.md#d14-how-do-events-route-to-targets)). The `temporal` target runs each matching rule's `start`, `signal`, or `signal_with_start` with a workflow ID from a template, such as `review:<repo>#<n>@<head>`, which follows the action key idea in [appendix B](B-earlier-designs.md). It passes a reference to the event, never its payload, and parks events it can't deliver ([D15](decisions.md#d15-how-does-the-temporal-target-deliver-an-event-and-what-does-it-do-with-one-it-cant-deliver)). Its durable consumer makes delivery at least once, and the workflow ID and the workflow's own `delivery` check make a repeat harmless.
- **Intent sessions:** the operator's small intent sessions write issues on GitHub. An `issues` event reaches the gateway and starts the matching workflow, routed on labels that sm owns, such as `role:<name>`. No new source is needed. A route on sm's Handler field isn't built ([D17](decisions.md#d17-may-the-gateway-hold-a-credential-that-reads-org-project-fields-so-rules-can-match-a-requests-handler)).
- **Chaining:** a workflow starts the next one by writing an issue, comment, or label to GitHub, which the normal loop picks up ([D16](decisions.md#d16-may-workflows-agent-runs-and-sm-publish-events-into-the-gateway)). A later build step could add internal publish.
- **Workers:** a worker runs activities that read the event by `seq`, call `tbhbdev alpha agent exec`, then `wait`, and record the result. tools changes the launcher to accept a named non-interactive caller.
- **Coordination sessions** stop once workflows cover their work.
- **Other targets:** a chat notice and a file sink are push targets behind the same `Target` interface, each one package and one config block, with their own rules. A second Temporal cluster is a second `temporal` instance with its own name, connection, and consumer. None of them changes the envelope, the subjects, or the interface.
