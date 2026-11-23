---
type: Decision Record
title: The changed paths and content set the review tier
description: Review tiers set how many runs a pull request needs, and a fix gets one review of its delta.
tags: [reviews, tiers, pull-requests]
status: stable
decided: "2026-11-23"
generated:
  - role: generated
    harness: codex:0.171.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:4f00750704d1bb6e
    at: "2026-11-23T02:44:49Z"
---

# The changed paths and content set the review tier

## Context

Review costs depend on what a change can affect. A prose edit, a test change and code that removes the only copy of work need different levels of scrutiny.

## Decision

The diff’s paths and content set the tier. Reviewers can raise it. Once set, it never falls.

- Tier 0 needs CI only for prose-only changes in wiki, draft, research and spike repositories, except wiki design documents.
- Tier 1 needs one fresh run for most code, tests, refactors and documentation in code repositories, wiki design documents and deletion of regenerable state.
- Tier 2 needs two fresh runs for changes to credentials, rulesets, permissions, agent harness configuration, deployments or the review and merge path. It covers destructive commands and code that can delete durable data or the only copy of work.

At every tier, a fix gets one fresh run on the delta.

## Consequences

The commissioning role merges a tier 0 pull request once its head is green. It needs no review, approval or separate merger assignment. Reviewers apply the tiers now, while the review loop design specifies the path lists.

Tier 2 examples include changes to `.mcp.json` and deletion of uncommitted changes, branches or worktrees with work that has yet to be merged or pushed, stashes, records and backups. Deleting build caches stays at tier 1.

A consumer-only pin bump to a commit already reviewed and merged upstream gets one run. A higher upstream tier applies only to code changed in the same pull request. Routine third-party version and checksum bumps stay under dependency maintenance and need only a CI signal.
