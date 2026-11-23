---
type: Decision Record
title: A rebuilt pull request can reuse approval for an identical tree
description: The review worker can approve a replacement without a new model read when the rebuilt Git tree matches the approved tree.
tags: [approvals, reviews, git]
status: stable
decided: "2026-11-12"
---

# A rebuilt pull request can reuse approval for an identical tree

## Context

A pull request sometimes needs rebuilding after its content already passed review. Repeating the same review by a model adds cost when the replacement contains the same files. A Git tree records the paths and contents at a commit.

## Decision

The review worker can approve a rebuilt pull request when its Git tree matches the earlier approved tree. Tree identity stands in for [reviewer evidence for the exact head and the independent approver’s read of the full diff](0009-the-reviews-app-posts-approvals-for-an-independent-approver.md). The [earlier verdict](0004-review-runs-publish-verdicts-on-the-pull-request.md) stays attached to the commit the reviewer read. The tree match is evidence for a new approval on the replacement’s head.

At the replacement’s head, the `conclusion` check must pass, GitHub must mark every listed commit’s signature as verified, and the service must reject approval from the opener’s root identity. The worker holds the replacement if it changes the review template, which sets the required format for review results.

## Consequences

Changing only the commit identifier leaves the file tree unchanged. Revoking this authorization restores a fresh model review for rebuilt replacements.
