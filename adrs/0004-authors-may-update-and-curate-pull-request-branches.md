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

With merge commits, authors can choose how to maintain a pull request branch. Every authored commit becomes part of the default branch's history, so its message must make sense on its own.

## Decision

Authors may merge the default branch into a pull request branch or rebase onto it. A force-push to a pull request branch requires `git push --force-with-lease`. Default-branch rulesets block every force-push.

Authors choose how many commits to keep. Each authored commit has a Conventional Commit subject of at most 72 characters that describes its change. The message check rejects work-in-progress markers, fixup, squash and amend prefixes, and generic subjects that omit what changed.

## Consequences

An author can reword a rejected message and push the rewritten branch with a lease. A rewrite changes the head and requires a new review at the applicable tier. Authors who want to curate their commits can do so before requesting review.
