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

Renovate produces routine dependency updates. A manual merge handoff delays them after the required checks and approval are complete.

## Decision

Renovate enables GitHub platform automerge and keeps its five-day release cooldown. GitHub merges each eligible update after required CI and approval. The update follows the review tier that applies to its diff. Moving a pin to an already reviewed and merged commit, on its own, is tier 1 and gets one review run.

For an update that runs a command with the Renovate token, the approval check also confirms that the changes are confined to dependency manifests and lockfiles.

## Consequences

The approval role handles successful CI events until a queue worker takes over automated approval. GitHub performs the merge once the requirements are met. The cooldown applies to release age and adds no waiting period after approval. Disabling automerge in the shared preset restores a separate merge step.
