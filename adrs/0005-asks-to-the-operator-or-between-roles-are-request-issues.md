---
type: Decision Record
title: Asks to the operator or between roles are Request issues
description: A request for a decision, an action, access or information is a Request issue, whether the operator or an agent handles it, and session messages stay for steering and urgent stops.
tags: [requests, issues, agents, work-tracking]
status: stable
decided: "2026-11-23"
---

# Asks to the operator or between roles are Request issues

## Context

Asks to the operator sat in a hand-kept list of open items with its own answer flow. Asks between agent roles went as session messages, which are lost when a session ends or compacts, can't be seen by anyone else, and can only be answered by the session that received them. That also forces long-running sessions, since nothing else holds the ask.

## Decision

An ask to the operator, or from one role to another, for a decision, an action, access or information is an issue of the org Request type in the private service-management records repository. The org Handler field says who picks it up: Operator or Agent. An agent Request carries an `area:<repo>` label, which scopes the context a run loads, and `cap:` labels suggesting the capabilities fulfilling it takes. The capability labels are hints for routing: the handling role or the router may relabel or reassign, no check refuses a Request over them beyond their being in the vocabulary, and routing never blocks on them. Every handler uses the same template, lifecycle and commands. Session messages stay for steering and urgent stops.

## Options considered

- Typed Handler values, one per agent role: rejected because an org issue field has a limit on its allowed values.
- Per-role or per-repository owner labels: rejected because agents look up the context each task needs, so the roles registry defines role kinds and capabilities with no owner per repository.

## Consequences

Asks become durable and visible, and any run of the right kind can answer one. Event delivery can route Request events to a launched run, which allows loops that need no long-running session. Stalled agent Requests are chased like stalled pull requests. The operator answers a Request with dedicated commands that post the answer from their own login. Until the Request commands and the agent-handler support land, the existing list of open items and its answer flow stay the live path for operator asks, and asks between roles still travel as messages. The old list retires at cutover.
