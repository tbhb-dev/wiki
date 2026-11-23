---
type: Decision Record
title: 0035. Change issues use three approval classes
description: Change issues come from change Requests and follow Standard, Normal, or Emergency approval rules.
tags: [service-management, changes, approvals]
status: stable
decided: "2026-11-22"
---

# 0035. Change issues use three approval classes

## Context

Repeatable work, planned changes that need approval, and emergency fixes need different approval timing and a common history.

The Change workflow governs how to document a change and when it needs approval or review, with [Normal Change approvers](0036-normal-changes-follow-the-scope-of-delegated-authority.md) set by the delegated authority rules.

## Decision

Create a Change issue to track work proposed in a [Request](0034-operator-and-agent-requests-use-github-issues.md). A Standard Change follows a repeatable procedure with advance approval. A Normal Change needs approval before the work starts. An Emergency Change can proceed immediately and receives review afterward.

Use the template for that approval class. Record the plan, the risk, and how to undo the change. For a Normal Change, record the approver under ADR 0036. Where ADR 0036 names no approver, the coordinator approves, except for a change the coordinator owns. The operator approves that change because nobody approves their own change. After the change, add evidence of the work and how you verified it.

## Consequences

Before work starts, a single-role, irreversible Normal Change outside the operator's reserved matters has a recorded approval from the coordinator, or from the operator when the coordinator owns the change. Change issues replace the temporary change logs when the workflow goes live. A pull request that brings an already deployed change back into the repository links to the Change issue.
