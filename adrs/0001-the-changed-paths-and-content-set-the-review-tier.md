---
type: Decision Record
title: The changed paths and content set the review tier
description: The files and content a pull request changes set how many reviews it needs.
tags: [reviews, tiers, pull-requests]
status: stable
decided: "2026-11-23"
---

# The changed paths and content set the review tier

## Context

Review costs depend on what a change can affect. Prose edits, test changes and code that deletes the only copy of work warrant different scrutiny.

A review tier sets the number of fresh review runs, started anew for the change. The diff shows the proposed file changes. Continuous integration (CI) runs automated checks. A spike repository holds exploratory work. Rulesets group rules that GitHub enforces. An agent harness runs an agent. Regenerable state can be rebuilt. Durable data needs to persist.

A worktree is a Git checkout. A stash stores uncommitted changes.

The tier sets the initial review count and leaves fixes to a separate rule that applies at every tier.

## Decision

The diff’s file paths and content set the tier. Reviewers can raise it. Once set, it never falls.

- Tier 0 needs CI only for prose-only changes in wiki, draft, research and spike repositories, except wiki design documents.
- Tier 1 needs one fresh run for most code, tests, refactors and documentation in code repositories, wiki design documents and deletion of regenerable state.
- Tier 2 needs two fresh runs for changes to credentials, rulesets, permissions, agent harness configuration, deployments or the review and merge path. It covers destructive commands and code that can delete durable data or the only copy of work.

A fix gets one fresh run on the changes made by that fix.

## Consequences

The commissioning role, which starts the build, merges tier 0 changes once checks pass on the head, the latest source commit. Tier 0 requires no review, approval or separate merger assignment. Reviewers apply the tiers now. The review loop design, which describes how reviews run, specifies the path lists.

Tier 2 covers changes to `.mcp.json`, the Model Context Protocol configuration for external tools. It also covers deletion of uncommitted changes, stashes, records, backups and branches or worktrees with work yet to merge or push. Deleting build caches stays at tier 1.

A change solely to a dependency pin, a reference to a specific dependency revision, gets one run if that commit was reviewed and merged at its source. A higher tier from the source repository applies only to code changed in the same pull request. Routine third-party version and checksum updates remain dependency maintenance. They need only CI.
