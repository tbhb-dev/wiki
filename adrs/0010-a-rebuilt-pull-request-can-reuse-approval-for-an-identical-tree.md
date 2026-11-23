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

The review worker, the service that handles queued reviews, can approve a rebuilt replacement without another model review if its Git tree matches the tree covered by the earlier approval. This rule covers reuse of prior approval and leaves the [approval checks](0009-the-reviews-app-posts-approvals-for-an-independent-approver.md) for the replacement’s current head in force. The head is the latest commit on the pull request’s source branch. The worker holds the replacement if it changes the review template, which sets the required format for review results.

## Consequences

Changing only the commit identifier leaves the file tree unchanged. Revoking this authorization restores a fresh model review for rebuilt replacements.
