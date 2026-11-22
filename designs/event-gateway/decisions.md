---
type: Design Appendix
title: Event gateway decisions
description: The open and decided questions of the event gateway design.
tags: [event-gateway, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Event gateway decisions

Back to [the design](README.md).

## Open

None.

## Decided

### D13. How do subscriptions follow one issue, pull request, or label?

- **Decided:** 2026-11-22, by the gateway maintainer, on the coordinator's request.
- **Rule:** add two optional subscription filters, `number` and `label`, for the GitHub events `issues`, `issue_comment`, `pull_request`, and `pull_request_review`. `label` matches the item's current labels and, on a `labeled` or `unlabeled` action, the changed label, so a follower of a label sees the event that removes it. Neither filter is required, and a filter on any other event is refused ([appendix C](C-protocol.md#session-protocol)).
- **Why:** sessions follow one work item or one role's label instead of every event in a repository, and only the gateway builds it because it replaces awrel.
- **Undo:** two-way door. The filters are optional and additive, so a client that never sends them is unaffected, and removing them breaks only subscriptions that use them.
- **Record:** this pull request.

### D12. What queues events inside the gateway?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** NATS with JetStream, embedded in the gateway process for now. One file-backed stream, `EVENTS`, on subjects `events.<provider>.<event>`, replaces the SQLite log, and durable consumers replace its cursor table ([appendix C](C-protocol.md#stream-configuration)).
- **Why:** a maintained queue replaces a homegrown log, cursor table, and sweep.
- **Undo:** two-way door. The `Target` interface and the stream subjects let a separate NATS server replace the embedded one without changing targets.
- **Record:** the operator's decision of 2026-11-22, written into this design by [wiki#72](https://github.com/tbhb-dev/wiki/pull/72), merged as `d19d86fd`.

### D11. How does a client send a subscription set larger than one frame?

- **Decided:** 2026-11-22, by the gateway maintainer.
- **Rule:** the client splits `hello` across frames of at most 4096 bytes each, marked with `more`, up to 256 subscriptions. The server delivers nothing until the last frame, and buffers live events from the first ([appendix C](C-protocol.md#session-protocol)).
- **Undo:** two-way door before step 6 ships. After it, a new subprotocol version would carry a different scheme.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`. It carries the revision that answered the design's first review.

### D10. What is the end state behind the gateway?

- **Decided:** by 2026-11-21, by the operator, before the design.
- **Rule:** Temporal on owned hosts runs workflows behind the gateway, workflows launch agents through `tbhbdev alpha agent exec`, and intent sessions replace coordination sessions.
- **Undo:** two-way door. The operator can change the end state, and the gateway's targets stay pluggable.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`.

### D9. Where does the gateway send telemetry?

- **Decided:** by 2026-11-21, by the operator, before the design.
- **Rule:** every service sends OpenTelemetry to the loopback-only local collector, and logs go to Loki.
- **Undo:** two-way door, by pointing the exporter elsewhere.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`.

### D8. May the gateway use paid services?

- **Decided:** by 2026-11-21, by the operator, before the design.
- **Rule:** no paid services. Public ingress uses a Cloudflare Tunnel.
- **Undo:** two-way door. The operator can lift the rule, and the ingress swaps without code changes.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`.

### D7. Is the gateway a new project that replaces awrel?

- **Decided:** by 2026-11-21, by the operator, before the design.
- **Rule:** tbhb-dev/event-gateway is a new public project, self-hosted on owned hosts, and the front door for event-driven work. It supersedes awrel, whose code stays unchanged until it retires.
- **Undo:** two-way door until step 10 retires awrel. After that, restoring awrel means redeploying it and adding its webhooks back.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`.

### D6. May tbhb-dev publish the gateway's container image as a public package?

- **Decided:** 2026-11-22, by the design owner, as a reversible call.
- **Rule:** no public image for now. ops builds the image on the host from a pinned commit, the same as the agent-capacity collector. Step 2 builds on the host from a pinned commit.
- **Undo:** publish to the registry and deploy by digest.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`.

### D5. Should the gateway's GitHub webhook be organization-wide?

- **Decided:** 2026-11-22, by the design owner, as a reversible call.
- **Rule:** yes. One organization-wide webhook for the events sessions subscribe to today, added beside awrel's. An organization owner creates it.
- **Why:** one webhook and one secret cover every repository, including new ones, and step 10 removes one webhook rather than many.
- **Undo:** per-repository webhooks, added and removed per repository.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`.

### D4. May the gateway hold a GitHub credential to request redelivery of failed webhooks?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** yes. A dedicated GitHub App with only the webhook-redelivery permission, installed on the organization at step 9. ops stores the App key in 1Password and delivers it as a file.
- **Why:** GitHub doesn't retry a failed delivery on its own. Without redelivery, an outage of the one host loses events, as awrel's disconnects do.
- **Undo:** uninstall the App.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`.

### D3. May the gateway keep raw event payloads at rest for 7 days?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** yes. Raw payloads stay on the owned host's volume only, never in logs or traces. Since D12 the stream's `MaxAge` deletes each whole message at 7 days, and backups of the volume, if any, follow the same window.
- **Why:** replay and the read API need payloads. Payloads from private repositories carry private content.
- **Undo:** store envelopes only, going forward. Data already copied stays copied.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`.

### D2. Should the session endpoint stay off public ingress?

- **Decided:** 2026-11-22, by the design owner, as a reversible call.
- **Rule:** yes. The session endpoint and the read API stay off public ingress, and the tunnel routes only the webhook paths. Sessions and the read API connect over the private network, so a session off it, such as a hosted cloud session, can't connect.
- **Why:** every session runs on an owned host, and awrel's public endpoint with one shared token is its largest exposure. github-metrics made the same call for its tunnel.
- **Undo:** add the tunnel route for `/connect` and the read API. Per-client tokens still apply.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`.

### D1. May the gateway use the Go module path `go.tbhb.dev/event-gateway`?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** yes. The gateway uses `go.tbhb.dev/event-gateway`, matching `go.tbhb.dev/awrel`. Step 0 adds the vanity entry.
- **Undo:** costly after the first release, since published versions keep the path.
- **Record:** [wiki#70](https://github.com/tbhb-dev/wiki/pull/70), merged as `69542e44`.
