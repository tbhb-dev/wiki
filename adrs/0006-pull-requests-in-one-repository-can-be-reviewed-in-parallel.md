---
type: Decision Record
title: Pull requests in one repository can be reviewed in parallel
description: Each qualifying head enters review as capacity permits, including heads in a stack of pull requests.
tags: [reviews, concurrency, pull-requests]
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

# Pull requests in one repository can be reviewed in parallel

## Context

A single review slot per repository can idle reviewers and delay independent changes. Stacks also let authors isolate a dependent change from the work it builds on.

## Decision

Several pull requests in one repository may be open and in review at the same time. The available review capacity and launcher limits determine how many runs start. Review each stacked pull request against its own base.

Merge commits preserve the parent’s commit IDs. After the parent merges, change the child’s base to the default branch. Review coverage carries over when its head and reviewed diff stay the same.

## Consequences

Owners can begin the next change before the current pull request merges. A change to the child’s own diff needs review, and rewriting its head calls for a verdict on the new head.
