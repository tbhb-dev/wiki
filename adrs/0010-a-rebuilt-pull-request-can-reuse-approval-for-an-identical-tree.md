---
type: Decision Record
title: A rebuilt pull request can reuse approval for an identical tree
description: A rebuilt pull request still needs a full-diff review at its exact head when its Git tree matches the approved tree.
tags: [approvals, reviews, git]
status: stable
decided: "2026-11-12"
---

# A rebuilt pull request can reuse approval for an identical tree

## Context

A Git tree records the paths and contents at a commit. Rebuilding a pull request can leave its tree unchanged while the base moves. The diff can therefore change even when the tree matches, so the review run reads the diff against the current base.

## Decision

A rebuilt pull request whose Git tree matches the approved tree still needs one review run at its exact head. The run reads the whole diff against the current base, confirms that the trees match, and may cite the [earlier verdict](0004-review-runs-publish-verdicts-on-the-pull-request.md). The earlier verdict stays attached to the commit the reviewer read. The new run gives its own verdict for the replacement’s head.

The [approval checks](0009-the-reviews-app-posts-approvals-for-an-independent-approver.md) still apply at the replacement’s head. The `conclusion` check must pass, GitHub must mark every listed commit’s signature as verified, and the service must reject approval from the opener’s root identity.

## Consequences

A re-commit adds one review run at the new head. The existing tier reviews still cover unchanged work. A tree match never replaces a review at a new head.
