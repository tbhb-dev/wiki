---
type: Decision Record
title: A workflow can approve pin-only Renovate pull requests
description: A reusable workflow and a dedicated GitHub App approve Renovate pull requests that only move pins and pass the required checks.
tags: [approvals, renovate, github-actions]
status: stable
decided: "2026-11-24"
---

# A workflow can approve pin-only Renovate pull requests

## Context

Repositories that require one approval make a person approve every Renovate pull request, even when the change only moves a pin and CI covers it. A pin is a version, tag, commit, or digest that a file names for a dependency. The [design](../designs/renovate-auto-approval/README.md) describes a reusable workflow in tbhb-dev/rules and a dedicated GitHub App, `tbhb-dev-renovate-approver`, that approve such pull requests. The first-party review rule normally runs a review for every change to a first-party pin.

## Decision

A pull request qualifies when the Renovate App authored it, a pure decision function finds that it changes only recognized pins and their companion lockfile entries, and every required check passed in the triggering CI run. The App's approval then satisfies the approval requirement. For a first-party pin, the approval also replaces the review run once the workflow verifies that the source repository's default branch contains the new commit. Every other pull request goes through normal review. Each decision logs a reason code.

## Scope

This applies to repositories that call the workflow. Each caller starts in `evaluate` mode, where the workflow logs its decision and posts nothing. Moving a caller to `enforce` follows the review in the design's rollout appendix. The code, the workflow, each caller, the App, and its environment configuration are tier 2 changes.

## Consequences

Pin-only updates merge without a person once a repository enforces. The matcher becomes part of the merge path, so a matcher bug could let a change that nobody reviewed merge. The design bounds that with a fail-closed grammar, tests for every reason code, an evaluate period, and two independent tier 2 reviews. The App holds a credential for every repository that calls the workflow.

## Confirmation

A reviewer checks that each caller names `mode`, that the reason-code tests exist, and that the rollout review links a decision-log review.
