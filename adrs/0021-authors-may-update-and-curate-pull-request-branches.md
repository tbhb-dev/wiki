---
type: Decision Record
title: Authors may update and curate pull request branches
description: Authors may merge in or rebase onto the default branch and use a lease for any force-push to a pull request branch.
tags: [branches, git, pull-requests]
status: stable
decided: "2026-11-22"
---

# Authors may update and curate pull request branches

## Context

A merge commit joins two branch histories and preserves every authored commit. Each such commit becomes part of the repository's primary branch, or default branch. Its message must make sense on its own. The rule gives authors control over pull request branch history and keeps history rewrites off the default branch.

## Decision

Authors may merge the default branch into a pull request branch or rebase it by replaying its commits onto the default branch. A push that replaces the remote branch's history must use `git push --force-with-lease`, which checks that the remote branch is still at the expected commit. Default-branch rulesets, GitHub's branch protections, block every force-push.

Authors choose how many commits to keep. Each commit subject describes its change in at most 72 characters using Conventional Commits, a format that prefixes the summary with a change type. The check rejects work-in-progress markers, Git's fixup, squash and amend prefixes for later edits, and generic subjects that omit what changed.

## Consequences

An author can reword a rejected message and push the rewritten branch with a lease. A rewrite changes the head, the branch's latest commit. It requires a new review at the tier, or review level, assigned to the changes. Authors can curate their commits before requesting review.
