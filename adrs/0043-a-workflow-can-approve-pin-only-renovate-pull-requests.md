---
type: Decision Record
title: 0043. A workflow can approve pin-only Renovate pull requests
description: A reusable workflow and a dedicated GitHub App approve Renovate pull requests that only move pins and pass the required checks.
tags: [approvals, renovate, github-actions]
status: stable
decided: "2026-11-23"
---

# 0043. A workflow can approve pin-only Renovate pull requests

## Context

Repositories that require one approval make a person approve every Renovate pull request, even when the change only moves a pin and the required checks pass. A pin is a version, tag, commit, or digest that a file names for a dependency. The [design](../designs/renovate-auto-approval/README.md) describes a reusable workflow in tbhb-dev/rules and a dedicated GitHub App, `tbhb-dev-renovate-approver`, that approve such pull requests. The first-party review rule normally runs a review for every change to a first-party pin.

## Decision

A pull request qualifies when the Renovate App authored it from a branch in the repository to the default branch, it is open, not a draft, and has no opt-out label, a pure decision function finds only recognized pins within the allowed bump levels and their companion lockfile entries, and every required check passed in the triggering CI run. Just before approving, the workflow rechecks the head, base, and checks and refuses if any changed. The workflow then posts the App's approval, and the design's spike verifies that GitHub counts it. For a first-party pin, the approval replaces the one review run that [0024](0024-renovate-pull-requests-automerge-after-approval-and-ci.md) requires, once the source repository's default branch contains the new commit. Others get normal review.

## Scope

This applies to repositories that call the workflow. Each caller starts in `evaluate` mode, where the workflow logs its decision and posts nothing. Moving a caller to `enforce` follows the review in the design's rollout appendix. The code, the workflow, each caller, the App, and its environment configuration are tier 2 changes.

## Consequences

Qualifying pin-only updates can merge without a person once a repository enforces, if GitHub counts the App's approval, which the design's spike verifies. The matcher becomes part of the merge path, so a matcher bug could let a change that nobody reviewed merge. The design bounds that with a fail-closed grammar, tests for every reason code, an evaluate period, and two independent tier 2 reviews. One App key serves every caller, held as a secret in each caller's environment.

## Confirmation

A reviewer checks that each caller sets `mode`, that the reason-code tests exist, and that the rollout review links a decision-log review.
