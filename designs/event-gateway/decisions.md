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

### D1. May the gateway use the Go module path `go.tbhb.dev/event-gateway`?

- **Recommendation:** yes.
- **Why:** it matches `go.tbhb.dev/awrel`, and sessions run the client with `go run`, so the path lands in every harness config.
- **If yes:** step 0 adds the vanity entry.
- **If no:** name another path before step 1.
- **Reversible:** costly after release, since published versions keep the path.

### D2. Should the session endpoint stay off public ingress?

- **Recommendation:** yes. The tunnel routes only the webhook paths, and sessions and the read API connect over the private network.
- **Why:** every session runs on an owned host, and awrel's public endpoint with one shared token is its largest exposure. github-metrics made the same call for its tunnel.
- **If yes:** a session off the private network, such as a hosted cloud session, can't connect.
- **If no:** add `/connect` and the read API to the tunnel route. Per-client tokens still apply.
- **Reversible:** yes, by changing the tunnel route.

### D3. May the gateway keep raw event payloads at rest for 7 days?

- **Recommendation:** yes, on the owned host's volume only, never in logs or traces, with a sweep that deletes them after 7 days.
- **Why:** replay and the read API need payloads. Payloads from private repositories carry private content.
- **If yes:** replay covers a week. Backups of the volume, if any, follow the same window.
- **If no:** store envelopes only. Replay then sends envelopes, and sessions fetch details from GitHub.
- **Reversible:** yes, going forward. Data already copied stays copied.

### D4. May the gateway hold a GitHub credential to request redelivery of failed webhooks?

- **Recommendation:** yes, a dedicated GitHub App with only the webhook permission it needs, installed on the organization, in step 9.
- **Why:** GitHub doesn't retry a failed delivery on its own. Without redelivery, an outage of the one host loses events, as awrel's disconnects do.
- **If yes:** ops stores the App key in 1Password and delivers it as a file.
- **If no:** an operator redelivers by hand from GitHub's delivery log after an outage.
- **Reversible:** yes, by uninstalling the App.

### D5. Should the gateway's GitHub webhook be organization-wide?

- **Recommendation:** yes, one organization webhook for the events sessions subscribe to today, added beside awrel's.
- **Why:** one webhook and one secret cover every repository, including new ones, and step 10 removes one webhook rather than many.
- **If yes:** an organization owner creates it, and every repository's events reach the owned host.
- **If no:** per-repository webhooks, added and removed per repository.
- **Reversible:** yes.

### D6. May tbhb-dev publish the gateway's container image as a public package?

- **Recommendation:** yes, to GitHub's container registry, pushed only from `main` with the workflow's own token, and deployed by digest.
- **Why:** the same question is open for agent-capacity. One answer should cover both.
- **If yes:** step 2 pulls by digest.
- **If no:** ops builds the image on the host from a pinned commit.
- **Reversible:** yes, by deleting package versions.

## Decided

### D11. How does a client send a subscription set larger than one frame?

- **Decided:** 2026-11-22, by the event gateway owner.
- **Rule:** the client splits `hello` across frames of at most 4096 bytes each, marked with `more`, up to 256 subscriptions. The server delivers nothing until the last frame, and buffers live events from the first ([appendix C](C-protocol.md#session-protocol)).
- **Undo:** two-way door before step 6 ships. After it, a new subprotocol version would carry a different scheme.
- **Record:** the revision of this design that answers review of commit `f02d805`.

### D10. What is the end state behind the gateway?

- **Decided:** by 2026-11-21, by the operator, before the design.
- **Rule:** Temporal on owned hosts runs workflows behind the gateway, workflows launch agents through `tbhbdev alpha agent exec`, and intent sessions replace coordination sessions.
- **Undo:** two-way door. The operator can change the end state, and the gateway's targets stay pluggable.
- **Record:** this design's first commit, `003bd65`.

### D9. Where does the gateway send telemetry?

- **Decided:** by 2026-11-21, by the operator, before the design.
- **Rule:** every service sends OpenTelemetry to the loopback-only local collector, and logs go to Loki.
- **Undo:** two-way door, by pointing the exporter elsewhere.
- **Record:** this design's first commit, `003bd65`.

### D8. May the gateway use paid services?

- **Decided:** by 2026-11-21, by the operator, before the design.
- **Rule:** no paid services. Public ingress uses a Cloudflare Tunnel.
- **Undo:** two-way door. The operator can lift the rule, and the ingress swaps without code changes.
- **Record:** this design's first commit, `003bd65`.

### D7. Is the gateway a new project that replaces awrel?

- **Decided:** by 2026-11-21, by the operator, before the design.
- **Rule:** tbhb-dev/event-gateway is a new public project, self-hosted on owned hosts, and the front door for event-driven work. It supersedes awrel, whose code stays unchanged until it retires.
- **Undo:** two-way door until step 10 retires awrel. After that, restoring awrel means redeploying it and adding its webhooks back.
- **Record:** this design's first commit, `003bd65`.
