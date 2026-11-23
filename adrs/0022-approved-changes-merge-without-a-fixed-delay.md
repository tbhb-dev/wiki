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

A proposed 24-hour wait on approved changes that widen permissions would leave reviewed work idle.

## Decision

An eligible pull request can merge as soon as its required checks and approvals are complete. This also applies to an approved change that widens permissions.

## Consequences

Review and change classification determine the scrutiny before merge. Reverts and drift reporting let maintainers respond to trouble after merge.
