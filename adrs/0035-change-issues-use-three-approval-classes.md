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

## Decision

Create an organization Change issue from a Request for a change. For a Standard Change, use a repeatable model with advance approval. Approve a Normal Change before carrying it out. An Emergency Change can proceed immediately, with review afterward.

Use the corresponding template. Record the plan, risk, and rollback, plus the approver for a Normal Change. After the change, add evidence of the work and how it was verified.

## Consequences

Change issues replace the interim ledgers when the workflow goes live. Any pull request that backports a live change links to the Change issue.
