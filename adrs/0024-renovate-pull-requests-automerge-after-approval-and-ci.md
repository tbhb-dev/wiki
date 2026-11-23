---
type: Decision Record
title: Renovate pull requests automerge after approval and CI
description: GitHub automerge lands Renovate pull requests once CI and the required approval are complete.
tags: [dependencies, merge, renovate]
status: stable
decided: "2026-11-11"
---

# Renovate pull requests automerge after approval and CI

## Context

Renovate is a bot that opens routine dependency updates. A manual merge handoff delays them after approval and continuous integration (CI), the automated checks on a change. The rule covers merging Renovate's dependency updates and excludes pull requests from other authors.

## Decision

Renovate enables GitHub automerge, which merges eligible updates after required CI and approval. Renovate keeps its five-day cooldown, the minimum release age for an update. The update follows the review tier, or review level, assigned to its changes. Changing only a pin, a fixed commit reference, to an already reviewed and merged commit is tier 1 and gets one review run.

If an update runs a command with Renovate's access token, approval also requires changes to stay within manifests that declare dependencies and lockfiles that record their resolved versions.

## Consequences

The approver handles successful CI notifications until a queue worker, a service that processes queued work, takes over automated approval. GitHub performs the merge once the requirements are met. The release cooldown adds no waiting period after approval. Disabling automerge in the preset shared by repositories restores a separate merge step.

The planned title checker adds a required input to `conclusion` on `pull_request` events `opened`, `edited`, `synchronize`, and `reopened`. The ruleset uses `conclusion` to block GitHub auto-merge and the merge button on a failed, skipped, or missing title result. [Commit types and scopes design](../designs/commit-conventions/D-checker.md#required-title-check)
