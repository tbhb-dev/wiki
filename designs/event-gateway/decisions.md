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

### D6. May tbhb-dev publish the gateway's container image as a public package?

- **Decided:** 2026-11-22, by the coordinator, as a reversible call.
- **Rule:** no public image for now. ops builds the image on the host from a pinned commit, the same as the agent-capacity collector. Step 2 builds on the host from a pinned commit.
- **Undo:** publish to the registry and deploy by digest.
- **Record:** coordinator call recorded 2026-11-22.

### D5. Should the gateway's GitHub webhook be organization-wide?

- **Decided:** 2026-11-22, by the coordinator, as a reversible call.
- **Rule:** yes. One organization-wide webhook for the events sessions subscribe to today, added beside awrel's. An organization owner creates it.
- **Undo:** per-repository webhooks, added and removed per repository.
- **Record:** coordinator call recorded 2026-11-22.

### D4. May the gateway hold a GitHub credential to request redelivery of failed webhooks?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** yes. A dedicated GitHub App with only the webhook-redelivery permission, installed on the organization at step 9. ops stores the App key in 1Password and delivers it as a file.
- **Undo:** uninstall the App.
- **Record:** operator answer recorded 2026-11-22.

### D3. May the gateway keep raw event payloads at rest for 7 days?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** yes. Raw payloads stay on the owned host's volume only, never in logs or traces, and a sweep deletes them after 7 days. Backups of the volume, if any, follow the same window.
- **Undo:** store envelopes only, going forward. Data already copied stays copied.
- **Record:** operator answer recorded 2026-11-22.

### D2. Should the session endpoint stay off public ingress?

- **Decided:** 2026-11-22, by the coordinator, as a reversible call.
- **Rule:** yes. The session endpoint and the read API stay off public ingress. The tunnel routes only the webhook paths, and sessions and the read API connect over the private network. A session off the private network, such as a hosted cloud session, can't connect.
- **Undo:** add the tunnel route for `/connect` and the read API. Per-client tokens still apply.
- **Record:** coordinator call recorded 2026-11-22.

### D1. May the gateway use the Go module path `go.tbhb.dev/event-gateway`?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** yes. The gateway uses `go.tbhb.dev/event-gateway`, matching `go.tbhb.dev/awrel`. Step 0 adds the vanity entry.
- **Undo:** costly after the first release, since published versions keep the path.
- **Record:** operator answer recorded 2026-11-22.
