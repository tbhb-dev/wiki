---
type: Decision Record
title: A rebuilt pull request whose tree matches an approved one is approved without a model read
description: The review and merge worker approves a replacement pull request with no new review run when its tree equals a tree an earlier review approved.
tags: [review, approval, merge-queue]
status: stable
decided: "2026-11-12"
---

# A rebuilt pull request whose tree matches an approved one is approved without a model read

## Context

Pull requests sometimes get rebuilt as replacements with the same content, for example to re-author or re-sign their commits. A verdict covers only the exact head it judged, so each replacement would otherwise wait for a fresh review of code a reviewer already read.

## Decision

The review and merge worker may approve a rebuilt replacement pull request with no model read when its tree equals the tree an earlier review approved.

A replacement whose difference from the approved version lies in the pull request template doesn't qualify, and it stays held for review.

## Consequences

Replacements with unchanged content merge without spending review quota. The comparison is of trees, so a change to any file needs a review. To undo this, revert the authorization in the review and merge queue design.
