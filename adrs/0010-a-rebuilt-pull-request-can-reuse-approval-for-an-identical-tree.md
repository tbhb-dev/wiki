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

A pull request sometimes needs rebuilding after its content already passed review. Repeating the same model read adds cost when the replacement contains the same files.

## Decision

The review worker can approve a rebuilt replacement without another model read if its Git tree equals the exact tree covered by the earlier approval. The worker holds cases where the rebuilt pull request changes the review template.

## Consequences

A new commit ID alone does not make the file tree different. The replacement still goes through the approval gates for its current head. Revoking this authorization restores a fresh model read for rebuilt replacements.
