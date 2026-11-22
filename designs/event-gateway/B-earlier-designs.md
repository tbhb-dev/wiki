---
type: Design Appendix
title: Event gateway earlier designs
description: What earlier designs offer the event gateway, and which of their constraints it drops.
tags: [event-gateway, alternatives, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Appendix B. Earlier designs

Back to [the design](README.md).

Three earlier private designs, and one related one, shaped this design. They're inspiration, not references. This appendix names what each offers and which of its constraints this design drops.

## Grafana alerts on the relay

**Offers:**

- Verify the signature over the raw body before parsing, with one secret per provider.
- The route sets `provider`, so a body can't impersonate another source.
- Run checks in a fixed order, then publish one event per alert.
- Build a deterministic delivery ID from the body and the alert's index, so a retry keeps its IDs and consumers can dedupe.
- Build the payload from an allowlist, and treat alerts as notice, never permission.
- Keep a second notification channel for every rule, and send the relay-is-down alert only to that channel.

**Drops:** the edge platform, no replay, and filters that work only for GitHub. The gateway can filter alerts on `scope` from day one.

## Relay replay

**Offers:** a sequence number on each event, a bounded window in local storage, and a client that sends its last sequence number on reconnect. It also names the privacy cost of payloads at rest, which [D3](decisions.md) takes up.

**Drops:** the 30-minute window, which an edge platform's storage limits set. An owned disk can keep a week.

## Review and merge queues

**Offers:**

- Events wake a worker, and a periodic reconcile sweep catches what events missed. The gateway's durable log makes the sweep a backstop, not the main path.
- State derives from reads, and only action keys and receipts persist.
- Action keys such as `review:<root>@<head>` dedupe work and let a newer head supersede an older one. A Temporal workflow ID can follow the same form.
- Capacity gates, one global stop switch, and an off, shadow, on rollout for each rule.

**Drops:** the service-account-on-one-host worker, the hand-built queues, and the no-polling policy. Temporal replaces the queues. The gateway serves a pull API, so consumers can catch up without polling GitHub.

## Agent routing

**Offers:** a pure function from request and inputs to a decision, hard filters before a fixed sort, explicit flags that always win, and a run record that keeps every candidate's verdict. The gateway's routing from events to workflows can follow the same pure, recorded shape.

**Drops:** the launcher flag and the roster inputs. Those stay in tools.

## afakt and other public repositories

A search of afakt and the other public tbhb-dev repositories for gateway material found no event gateway ideas. awrel's own relay protocol is the closest public material, and [appendix C](C-protocol.md) builds on it.
