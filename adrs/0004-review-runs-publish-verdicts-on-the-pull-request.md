---
type: Decision Record
title: Review runs publish verdicts on the pull request
description: Reviewers publish their own results for the exact head, and approvers and mergers verify them at the source.
tags: [reviews, verdicts, pull-requests]
status: stable
decided: "2026-11-23"
generated:
  - role: generated
    harness: codex:0.171.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:4f00750704d1bb6e
    at: "2026-11-23T02:44:49Z"
---

# Review runs publish verdicts on the pull request

## Context

A relayed verdict can lose a finding or obscure which revision the reviewer read. A sizing result can also attach to the wrong revision if the head moves while the review is running.

## Decision

The review run posts a comment review on the pull request. It records the run identifier, provider, exact head, verdict of GO or HOLD, and any findings. The sizing reviewer posts the `size-review` status on the head it checked. The approver reads the result at its source and retains responsibility for approval. The merger confirms the verdict from the reviewer’s own output.

A verdict covers the exact head the run reviewed.

## Consequences

Until review runs can post to GitHub, the commissioning role posts the output unchanged with the run identifier and a reference to its log. The approver and merger use the original log to verify the verdict.
