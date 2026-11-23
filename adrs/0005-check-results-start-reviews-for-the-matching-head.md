---
type: Decision Record
title: Check results start reviews for the matching head
description: A successful conclusion check starts a queued review, and an isolated size failure starts a sizing review.
tags: [reviews, checks, events]
status: stable
decided: "2026-11-22"
generated:
  - role: generated
    harness: codex:0.171.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:4f00750704d1bb6e
    at: "2026-11-23T02:44:49Z"
---

# Check results start reviews for the matching head

## Context

An owner’s message about a passing check can be late or describe an earlier head. A pull request that only needs a sizing decision can also wait indefinitely for a passing check.

## Decision

Queue reviews against the head to be reviewed. A successful `check_run` event for the `conclusion` check releases a queued review when the event’s commit matches that head. When the size step is the only failing check, start the sizing review as soon as that result is available.

## Consequences

Owners can rely on check events to start reviews. Reverting the event trigger restores owner notifications, and reverting the sizing trigger restores an explicit sizing request.
