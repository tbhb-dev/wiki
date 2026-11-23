---
type: Decision Record
title: 0037. Service issue comments are append-only work notes
description: Structured work notes record updates, evidence, and state transitions on service issues.
tags: [service-management, work-notes, issues]
status: stable
decided: "2026-11-22"
---

# 0037. Service issue comments are append-only work notes

## Context

[Service issues](0033-service-workflows-run-on-github-issues.md) need a history of updates and evidence that explains how the issue reached its current state.

The work-note rule governs comments and state changes on service issues and leaves comments on other issues outside its scope.

## Decision

Use the service-management `note add` command to post a work note, a structured comment on a service issue. The work-note template records the author's role, the note's purpose, and the text with evidence for its claims. Note kinds cover progress updates, supporting evidence, transfers of responsibility, obstacles, decisions, and answers.

Append-only notes keep earlier entries as written. Add a new note that refers to the earlier one when making a correction. For every state transition, add a note that records who made the change, the previous state, the new state, and the reason.

## Consequences

The issue preserves the sequence of updates and corrections. Readers can trace a transition to its reason without reconstructing it from the current state.
