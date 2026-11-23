---
type: Decision Record
title: Pull request cycle time starts with the oldest commit trailer
description: Pull request cycle time runs from the earliest start recorded in its commit identity trailers to its merge or unmerged close.
tags: [cycle-time, metrics, pull-requests]
status: stable
decided: "2026-11-22"
---

# Pull request cycle time starts with the oldest commit trailer

## Context

Pull request timing measures elapsed time from when work began, which can precede the first commit or the opening of the pull request. A change can include work from multiple runs.

## Decision

Cycle time starts at the oldest `started_at` value in the pull request's commit identity trailers, including Generated-by and Assisted-by. It ends at the merge time or, for a pull request closed without merging, the close time. Metrics report unmerged closes as a separate state.

## Consequences

Until the metrics integration can list commits through read-only access, the pull request creation command carries the oldest commit-trailer value into the pull request body. A long-lived run can record a start earlier than its work on the change began, which makes the measured cycle time longer.
