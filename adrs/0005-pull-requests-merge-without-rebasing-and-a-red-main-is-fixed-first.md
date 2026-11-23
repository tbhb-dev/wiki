---
type: Decision Record
title: Pull requests merge without rebasing, and a red main is fixed first
description: A pull request needn't be up to date with main to merge; each repository's main is watched, and a red main is fixed by pull request ahead of other work.
tags: [merging, ci, main-branch, renovate]
status: stable
decided: "2026-11-22"
---

# Pull requests merge without rebasing, and a red main is fixed first

## Context

Two dependency lockfile bumps, each green on its own, merged back to back and broke a repository's main. Requiring every pull request (PR) to be rebased onto the latest main and tested again before merge would catch that case, at the cost of a rebase and a new CI run for every open PR whenever main moves.

## Decision

A PR doesn't have to be up to date with main to merge. Renovate rebases only when a PR conflicts (`rebaseWhen: "conflicted"`), and no rule forcing rebases is built.

What matters is knowing when a merge breaks main, and then fixing it. Each repository's default branch is watched through the event relay for a failed `conclusion` check. A red main is the first job of whoever owns that repository, fixed by PR ahead of other work. The coordinating role lists every red main in each standup summary.

## Options considered

- Require each PR to be up to date with main before merge: not adopted, because detecting a break and fixing it quickly costs less than rebasing every open PR.

## Consequences

A main can be red for the time it takes to notice and fix it. The fix lands by PR like any other change, never by a force-push.

A PR branch may still update by merging main in or by rebasing, as the merge-commit record allows. Watching falls to the repository's owning role today. When per-repository owners give way to runs launched per task, the watch moves with the work. Undo: a later decision.
