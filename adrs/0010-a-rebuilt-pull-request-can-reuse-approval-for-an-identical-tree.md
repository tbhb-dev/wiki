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

The review worker, the service that handles queued reviews, can approve a rebuilt replacement when its Git tree matches the tree covered by the earlier approval. The match replaces the fresh model review and satisfies the earlier verdict’s [revision limit](0004-review-runs-publish-verdicts-on-the-pull-request.md) by establishing that the reviewed tree is unchanged.

For the replacement’s exact head, the independent approver reads the full diff, confirms passing CI and uses the tree-identity result in place of reviewer evidence. All other [approval checks](0009-the-reviews-app-posts-approvals-for-an-independent-approver.md) apply at that head, the latest commit on the pull request’s source branch. The worker holds the replacement if it changes the review template, which sets the required format for review results.

## Consequences

Changing only the commit identifier leaves the file tree unchanged. Revoking this authorization restores a fresh model review for rebuilt replacements.
