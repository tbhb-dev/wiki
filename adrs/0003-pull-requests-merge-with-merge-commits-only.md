---
type: Decision Record
title: Pull requests merge with merge commits only
description: Every tbhb-dev repository merges with merge commits only, stacked pull requests work natively, and authors may force-push with lease to clean up a PR branch.
tags: [git, merging, pull-requests, stacking]
status: stable
decided: "2026-11-22"
---

# Pull requests merge with merge commits only

## Context

Squash merges rewrite a pull request's (PR's) commits. A PR stacked on another went stale when its parent merged and had to be recreated, which cost a fresh review. Banning stacked PRs and force-pushes worked around that, but serialized dependent work.

## Decision

Merges use merge commits only, and squash and rebase merging are off in every repository. The merge commit's title is `<PR title> (#N)` and its body is the cleaned PR body, never GitHub's default merge message. `tbhbdev alpha merge` sets both and refuses the default form, and repository settings back it up.

Stacked PRs work natively. When a parent merges, its commits keep their SHAs, and the child's base moves to main with its head unchanged, so its reviews stay valid.

An author may force-push with lease to a PR branch to clean up its commits before merge. Nothing requires it, and no check refuses a PR for its number of commits. A branch may update by merging main in or by rebasing onto main and force-pushing with lease. Force-pushes to main or any default branch stay blocked for everyone, and plain `--force` without lease stays out.

Every commit on a PR branch lands on main, so each needs a good message. A PR check refuses subjects not in Conventional Commits form, subjects over 72 characters, `fixup!`, `squash!`, `amend!` and WIP subjects, and bare generic subjects. An author fixes a refused message by rewording it and force-pushing with lease.

## Options considered

- Squash merges with no stacked PRs: rejected because dependent work waited or was recut and reviewed again.
- Merge commits with mandatory cleanup by force-push: rejected because curating commits stays the author's choice.

## Consequences

A verdict covers only the exact head it judged. A force-push makes a new head that needs review, so authors who clean up do it before they request review.

Signed commits, hook checks with no skipping, secret scanning and commit trailers apply to every authored commit. The commit identity check accepts merge commits while still checking authored ones.

Rulesets allow only the merge method and drop the linear-history requirement. A repository without a main ruleset can keep squash on until it gets one. Undo: turn squash back on and restore the force-push block.
