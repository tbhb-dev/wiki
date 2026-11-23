---
type: Decision Record
title: 0037. Service issue comments are append-only work notes
description: Structured work notes record updates, evidence, and state transitions on service issues.
tags: [service-management, work-notes, issues]
status: stable
decided: "2026-11-22"
generated:
  - role: generated
    harness: codex:0.166.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:15c30f9d922d6894
    at: "2026-11-23T02:46:10Z"
---

# 0037. Service issue comments are append-only work notes

## Context

Service issues need a history of updates and evidence that explains how the issue reached its current state.

## Decision

Post service issue comments through the service-management `note add` command. The template records the author role, the note kind, and the body with evidence for its claims. Note kinds cover updates, evidence, handoffs, blockers, decisions, and answers.

Work notes are append-only. A correction is a new note that refers to the earlier one. Every state transition adds a note with the actor, the previous state, the new state, and the reason.

## Consequences

The issue preserves the sequence of updates and corrections. Readers can trace a transition to its reason without reconstructing it from the current state.
