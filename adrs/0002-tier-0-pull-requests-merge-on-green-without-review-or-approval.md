---
type: Decision Record
title: Tier 0 pull requests merge on green without review or approval
description: The session that commissioned a tier 0 pull request merges it once its head is green, with no review, approval or named merger.
tags: [review, merge, tiers]
status: stable
decided: "2026-11-23"
---

# Tier 0 pull requests merge on green without review or approval

## Context

Under [0001](0001-review-depth-follows-tiers-set-by-the-paths-a-diff-touches.md), a tier 0 pull request gets no review run beyond CI. The merge process still asked for an approval and a separate go that named who would merge. With no review to approve, those steps only added a wait.

## Decision

A tier 0 pull request needs no review and no approval. The session that commissioned it merges it with the tbhbdev merge command once its head is green. No separate go names a merger.

This applies to tier 0 only. Tier 1 and tier 2 pull requests keep the approval and the named-merger step.

## Consequences

Prose changes in the wiki, drafts, research and spikes repositories land as soon as CI passes. A reviewer who raises a pull request out of tier 0 brings back the review, approval and merger steps for it. To undo this call, require a named merger for tier 0 again.
