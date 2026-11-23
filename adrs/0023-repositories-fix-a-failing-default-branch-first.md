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

Two pull requests can each pass continuous integration (CI), the automated checks on a change. Merging both in succession can still break the repository's primary branch, called the default branch. Prompt detection lets each change move independently. The rule covers rebasing before merge and repair priority after default-branch failures, leaving other merge conditions outside its scope.

## Decision

A pull request may merge without a rebase, which replays its commits onto the latest default branch. Renovate, the dependency update bot, rebases when its branches conflict with their target branch. The repository maintainer watches the default branch's CI result. A failure becomes the maintainer's first job. The fix goes through a pull request ahead of other work. The coordinator, who tracks work across repositories, reports failing default branches in every standup summary, the team's regular progress report.

## Consequences

The merge policy allows changes that may fail only in combination. Default-branch CI results and failure notifications tell maintainers when to act.
