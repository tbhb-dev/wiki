---
type: Decision Record
title: 0042. Project management defines the format for review verdicts
description: Project management maintains the templates and mechanical checks for review verdicts in the shared content module.
tags: [reviews, content, projects]
status: stable
decided: "2026-11-23"
---

# 0042. Project management defines the format for review verdicts

## Context

An approver, the role that accepts a revision for merge, needs to know which commit a review covers. Each finding needs to show how it affects the verdict, the review's pass or fail result.

This rule covers formats and checks for reviews and pull request descriptions and leaves approval authority to the existing review and merge rules.

## Decision

[Project management](0031-work-issues-live-in-the-repositories-responsible-for-them.md) maintains the review submission templates and format checks in [`tbhbdev.content`](0038-managed-content-writes-use-a-shared-command-path.md). Record the verdict, the exact commit reviewed, and the review tier, a numbered level from 1 to 3.

Mark findings that prevent a passing review with `BLOCK`. Mark deferred maintenance work with `TECH-DEBT`. Mark deferred security work with `SECURITY-DEBT`.

## Consequences

Approvers can read the same fields and finding categories on every review. Templates and checks for pull request descriptions also move into the shared content module.
