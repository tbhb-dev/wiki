---
type: Decision Record
title: Repositories fix a failing default branch first
description: Pull request branches can merge without rebasing, and a failing default branch takes priority over other work.
tags: [branches, ci, merge]
status: stable
decided: "2026-11-22"
---

# Repositories fix a failing default branch first

## Context

Two pull requests can each pass CI and still break the default branch when they merge in succession. Detecting that failure promptly keeps the two changes moving while they are independent.

## Decision

A pull request may merge without rebasing onto the latest default branch. Renovate rebases when its branches conflict. The role responsible for repository maintenance watches the default branch's CI conclusion. A failure becomes that role's first job, and the fix goes through a pull request ahead of other work. The coordinating role includes failing default branches in each standup summary.

## Consequences

The merge policy permits failures that show up only after changes combine. Default-branch CI and its failure events tell maintainers when to act.
