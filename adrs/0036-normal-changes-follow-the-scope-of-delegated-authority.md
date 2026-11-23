---
type: Decision Record
title: 0036. Normal Changes follow the scope of delegated authority
description: The responsible role, the coordinator, or the operator handles a Normal Change according to its scope and the authority it requires.
tags: [service-management, changes, approvals]
status: stable
decided: "2026-11-22"
---

# 0036. Normal Changes follow the scope of delegated authority

## Context

[Normal Changes](0035-change-issues-use-three-approval-classes.md) need an approval path that accounts for how far each role's authority extends. The constitution sets the limits on delegated authority. The Laws govern information exposure and authority over decisions.

This rule assigns approval authority for Normal Changes and leaves Standard and Emergency Changes to their own approval classes.

## Decision

The responsible role decides and records reversible work within its authority. The coordinator, the role responsible for cross-role matters, approves changes that cross roles. Send matters reserved by the constitution to the operator, who holds final authority, as [Requests](0034-operator-and-agent-requests-use-github-issues.md). Reserved matters include changes to the Laws. They include creating or expanding a kind of authority over decisions. They also include destroying the last working copy or permanently destroying anything the organization did not create.

Record each approval's basis under the constitution. Cite the role definition or recorded decision granting authority. Record discretion when the rules leave the decision to the role.

## Consequences

The record shows why the responsible role had authority to make a routine decision. Cross-role approval applies to the change itself, and reserved questions follow the operator Request workflow.
