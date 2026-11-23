---
type: Decision Record
title: The role that commissions a build can launch its reviews
description: The role that starts a build can also launch its reviews.
tags: [reviews, roles, independence]
status: stable
decided: "2026-11-23"
---

# The role that commissions a build can launch its reviews

## Context

The commissioning role starts the build run, the agent run that produces the change. That role has enough context to route the review. The build run has a stake in the result.

## Decision

The commissioning role can start new, separate review runs under the [review provider rules](0002-reviewers-come-from-a-different-provider-from-the-builder.md). The authority to start these runs stays outside the build run. This authority covers starting reviews and leaves judgment to the review runs, which issue verdicts and classify the issues they find. The commissioning role preserves the reviewers’ verdict text.

## Consequences

A change can move straight from build to review without a separate launcher. The approver, who decides whether to approve the pull request, reads the reviewers’ own verdicts.
