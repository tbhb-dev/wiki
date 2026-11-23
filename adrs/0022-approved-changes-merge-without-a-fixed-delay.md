---
type: Decision Record
title: Approved changes merge without a fixed delay
description: An eligible pull request can merge as soon as its required checks and approvals are complete.
tags: [merge, pull-requests, review]
status: stable
decided: "2026-11-11"
---

# Approved changes merge without a fixed delay

## Context

A proposed 24-hour wait on approved changes that widen permissions would leave reviewed work idle. A change widens permissions when it lets an account do more. The rule covers how soon an eligible pull request can merge and leaves the checks and approvals required for eligibility in place.

## Decision

A pull request that meets the repository's other merge requirements can merge as soon as its required checks and approvals are complete. This also applies to an approved change that widens permissions.

## Consequences

The policy category assigned to the change and the review determine the scrutiny before merge. Maintainers can respond to trouble after merge with reverts, which undo changes. Drift reports alert maintainers when the actual state departs from the expected state.
