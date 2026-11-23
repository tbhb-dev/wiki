---
type: Decision Record
title: The role that commissions a build can launch its reviews
description: The role that commissions a build can start its review runs and leaves the verdicts to those runs.
tags: [reviews, roles, independence]
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

# The role that commissions a build can launch its reviews

## Context

The role that starts a build has enough context to route its review. The build run itself produced the change and has a stake in the result.

## Decision

The commissioning role can launch fresh review runs in a context separate from the build, using the provider rules for review. The authority to launch reviews belongs to the commissioning role and stays outside the build run. The review runs alone judge the change, issue verdicts and classify findings. The commissioning role preserves the reviewers’ verdict text.

## Consequences

A change can move straight from build to review without a separate launcher. The approver reads the reviewers’ own verdicts.
