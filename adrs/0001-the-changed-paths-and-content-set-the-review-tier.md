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

The paths and content of the diff determine its tier. A reviewer can raise the tier. Once set, it can only stay the same or rise.

- Tier 0 needs CI only. It covers prose-only changes in wiki, draft, research and spike repositories, except for design documents in the wiki.
- Tier 1 needs one fresh review run. It covers most code, tests, refactors and documentation in code repositories, and design documents in the wiki.
- Tier 2 needs two fresh review runs. It covers changes to credentials, rulesets, permissions, agent harness configuration such as `.mcp.json`, deployments and the review and merge path. It also covers destructive commands and code that can remove durable data or the only copy of work. Examples include uncommitted changes, branches or worktrees with work that has yet to be merged or pushed, stashes, records and backups.

Deleting regenerable state such as build caches stays at tier 1. A consumer-only pin bump to a commit already reviewed and merged upstream also gets one run. The higher tier of any upstream code applies where that code changed. A routine third-party version and checksum bump falls under tier 1 dependency maintenance.

At every tier, a fix gets one fresh run scoped to the delta.

## Consequences

The commissioning role merges a tier 0 pull request once its head is green. It needs no review, approval or separate merger assignment. Reviewers apply the tiers now, while the review loop design specifies the path lists.
