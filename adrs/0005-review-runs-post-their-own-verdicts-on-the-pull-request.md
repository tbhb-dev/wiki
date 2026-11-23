---
type: Decision Record
title: Review runs post their own verdicts on the pull request
description: Each review run posts its verdict on the pull request for an exact head, and whoever approves or merges reads it there, never from a relay.
tags: [review, verdicts]
status: stable
decided: "2026-11-23"
---

# Review runs post their own verdicts on the pull request

## Context

Verdicts used to reach the approver through the commissioning session or a review-coordinating session. Every relay could drop, shorten or misread a finding, and the approver couldn't tell what the run itself had said.

## Decision

A review run posts its verdict directly on the pull request as a comment review. The review names the run, its provider and the exact head it judged, and gives GO or HOLD with the findings. A sizing review run posts the `size-review` status on the exact head itself.

A verdict covers only the exact head it judged. A new head, including one made by a force-push, needs its own review.

Approval stays the approver's. The approver, and any merger, confirms each verdict from the review run itself, never from a relay.

## Consequences

Verdicts no longer pass through the commissioning session or a review-coordinating session. The project-management tooling owns the template and mechanical checks for what review runs post: verdict, head, tier and findings.

Until review runs have their own write path, the commissioning session posts each run's output verbatim, with the run's name and the path to its log. The approver checks that post against the run log before relying on it.
