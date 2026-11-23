---
type: Decision Record
title: The session that commissions a build may launch its reviews but never judges them
description: The maker is the build run, so the session that commissioned it may launch fresh review runs, but it never writes, reclassifies or edits a verdict.
tags: [review, roles]
status: stable
decided: "2026-11-23"
---

# The session that commissions a build may launch its reviews but never judges them

## Context

The maker of a change never picks its own reviewer. An earlier reading treated the session that commissioned a build as its maker, so owner sessions could never launch their own reviews. That routed every review through a separate session and added extra review pairs to cover the gap.

## Decision

The maker is the build run, not the session that commissioned it. A session that commissioned a build may launch that change's review runs. Each review run is fresh, shares none of the build's context, and uses a provider other than the build's, per [0003](0003-reviews-come-from-a-provider-other-than-the-builders.md).

The commissioning session never judges the change. It writes no verdicts, reclassifies no findings and edits no verdict text. The approver reads the review runs' verdicts directly.

## Consequences

Owner sessions launch reviews without waiting on a review-coordinating session. A verdict reaches the approver without passing through the session that wants the change merged, as [0005](0005-review-runs-post-their-own-verdicts-on-the-pull-request.md) sets out.
