---
type: Decision Record
title: Changes follow three ITIL classes, and approval follows the constitution
description: A Change is a Standard, Normal or Emergency change with a plan, risk, rollback and verification, and the owning role approves it unless it crosses roles or touches a reserved matter.
tags: [changes, service-management, approvals]
status: stable
decided: "2026-11-22"
---

# Changes follow three ITIL classes, and approval follows the constitution

## Context

Changes to running systems, such as a tailnet grant, a ruleset edit or a hotfix applied by hand, were recorded in several direct-first ledgers kept per host, per hotfix and by the administrator role. Each ledger had its own shape, and nothing tied a hand-applied change to the pull request that backported it. The question of who approves a change also needed an answer that doesn't send every change to the operator.

## Decision

A Change is an org issue type, created from a change-kind Request. It belongs to one of three ITIL classes:

- **Standard** changes are pre-approved and repeatable, such as deploying merged code, applying settings, or creating a repository with the bootstrap command. Each standard change model is a template under `templates/standard-changes/`.
- **Normal** changes need an approver. Examples are tailnet grants, ruleset or bypass changes, a new GitHub App or permission, auto-mode rules and credential rotation.
- **Emergency** changes are hotfixes done first and reviewed after.

Each Change carries a plan, its risk, a rollback, an approver for a normal change, the implementation evidence and a verification. Its backport pull request links back to it.

Approval follows the constitution, with requests reaching the operator as rarely as possible. The owning role decides and records a reversible change, and the coordinator approves a change that crosses roles. Only the constitution's reserved matters go to the operator as a Request: changing the Laws, irreversible destruction of anything tbhb-dev didn't create, destroying the last working copy of anything, and adding or widening authority over decisions, such as roles, approvals, merge rights or the means to stop agents. Every approval names its authority: a role definition, a recorded decision, or the rules' silence.

## Consequences

The Change replaces the direct-first ledgers, so every change to a running system has one record with the same shape. The operator sees only reserved matters. A role that approves its own reversible change still records it, so the record is the check. Until the Change type and its templates are live, the existing ledgers stay the record.
