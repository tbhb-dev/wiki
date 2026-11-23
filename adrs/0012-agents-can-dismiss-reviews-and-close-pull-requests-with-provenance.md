---
type: Decision Record
title: Agents can dismiss reviews and close pull requests with provenance
description: Agents can dismiss GitHub reviews and close pull requests when the action records its provenance.
tags: [reviews, provenance, pull-requests]
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

# Agents can dismiss reviews and close pull requests with provenance

## Context

A stale review may need dismissal and an abandoned pull request may need closure. For either action, a later reader needs to know where the action came from.

## Decision

Agents can dismiss reviews and close pull requests through the GitHub broker. Each dismissal or close records its provenance.

## Consequences

Dismissal and closure become available through an auditable write path. The provenance record lets a maintainer trace who performed either action.
