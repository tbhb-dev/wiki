---
type: Decision Record
title: Review runs publish verdicts on the pull request
description: Reviewers publish their own results for the specific commit they examined.
tags: [reviews, verdicts, pull-requests]
status: stable
decided: "2026-11-23"
---

# Review runs publish verdicts on the pull request

## Context

A relayed verdict can omit an issue the reviewer found. It can also obscure which revision the reviewer read. A sizing review checks the size of a proposed change. Its result can attach to the wrong commit if the head, the latest commit on the source branch, moves during review.

## Decision

The review run posts a GitHub comment review, which reports results without approving or formally requesting changes. It records the identifier for the run, the model provider, the exact head, a verdict and any findings. GO means the reviewer clears the change. HOLD means the reviewer withholds clearance. The sizing reviewer posts its result as `size-review` on the head it checked.

A verdict applies to the exact head the run reviewed and excludes every other revision. The [approver](0009-the-reviews-app-posts-approvals-for-an-independent-approver.md) reads the result at its source. The approver remains responsible for approval. The merger, who merges the pull request, confirms the verdict from the reviewer’s own output.

## Consequences

Until review runs can post to GitHub, the [commissioning role](0003-the-role-that-commissions-a-build-can-launch-its-reviews.md) posts the output unchanged with the run identifier and a reference to its log. The approver and merger use the original log to verify the verdict.
