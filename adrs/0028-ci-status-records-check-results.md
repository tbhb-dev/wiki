---
type: Decision Record
title: CI status records check results
description: Pull request bodies, issue bodies and acceptance criteria leave CI results and CI-enforced items to the checks themselves.
tags: [acceptance-criteria, ci, pull-requests]
status: stable
decided: "2026-11-22"
---

# CI status records check results

## Context

Continuous integration (CI) runs automated checks and records their status on the checked commit. A result copied into prose can become stale when the pull request changes. A checklist that repeats CI items gives contributors a second place to maintain the same information. The rule covers check claims in pull request descriptions, issue descriptions and acceptance criteria while leaving those fields available to describe the change and its required behavior.

## Decision

CI status is the record for automated checks. Pull request descriptions, issue descriptions and acceptance criteria omit claims about tests, lint checks for code and style problems, and coverage of code by tests. They also omit results of required checks and checklists of items that CI already enforces. The [definition of done](0027-tbhb-dev-has-one-definition-of-done.md), the shared completion standard, labels the items CI enforces. Issue acceptance criteria exclude those items.

## Consequences

The [pull request draft check](https://github.com/tbhb-dev/tools/blob/main/docs/reference/alpha/commands.md) enforces this rule before submission. The [shared checks for outgoing GitHub content](0018-agents-use-tbhbdev-for-github-operations.md) also enforce it. Authors use descriptions and acceptance criteria to state what the change must do. Readers use the check results on the relevant commit for CI evidence.
