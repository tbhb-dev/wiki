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

A check result copied into prose can become stale when the pull request changes. A checklist that repeats CI items gives contributors a second place to maintain the same information.

## Decision

CI status is the record for automated checks. Pull request bodies, issue bodies and acceptance criteria omit test, lint and coverage claims, gate results, and checklists of items that CI already enforces. The definition of done marks which items CI enforces and keeps them out of issue acceptance criteria.

## Consequences

The pull request draft check and the shared content checks enforce this rule. Authors use the body and acceptance criteria to describe the change and its required behavior. Readers use the check results on the relevant commit for CI evidence.
