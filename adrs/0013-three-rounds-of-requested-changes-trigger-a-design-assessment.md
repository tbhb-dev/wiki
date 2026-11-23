---
type: Decision Record
title: Three rounds of requested changes trigger a design assessment
description: Repeated requests for changes prompt an assessment of the design and requirements without an automatic handoff to a human.
tags: [reviews, escalation, design]
status: stable
decided: "2026-11-11"
generated:
  - role: generated
    harness: codex:0.171.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:4f00750704d1bb6e
    at: "2026-11-23T02:44:49Z"
---

# Three rounds of requested changes trigger a design assessment

## Context

Successive rounds of fixes can reveal that the design or requirements are wrong. Sending every pull request to a human at a fixed count misses that distinction.

## Decision

After three rounds of requested changes on a pull request, assess whether the design or requirements need revision. The count triggers the assessment. A human handoff remains a separate decision.

## Consequences

The escalation path needs to carry the outcome of the assessment. The threshold alone does not settle who takes over the work.
