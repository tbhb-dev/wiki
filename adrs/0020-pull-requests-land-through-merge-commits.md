---
type: Decision Record
title: Pull requests land through merge commits
description: GitHub lands pull requests with merge commits so dependent pull requests keep their commit history.
tags: [git, merge, pull-requests]
status: stable
decided: "2026-11-22"
---

# Pull requests land through merge commits

## Context

A squash merge replaces a pull request branch's commits with one commit. If another pull request builds on that branch, its diff can show the original changes again. A diff compares the proposed changes with the branch they target. The author may need to rebuild the branch. The rule governs how repositories merge pull requests and format their merge messages, leaving preparation of individual commits to their authors.

## Decision

Repositories enable only merge commits, which join the source and target histories without replacing their commits. The first line of the merge message uses the pull request title with its number. The rest uses the pull request description after the merge command formats it for a commit message. Repository settings reinforce the command's message rules.

Record 0006 covers stacked pull requests, which build on other pull request branches. Record 0014 covers `Merged-by`, the commit message field that identifies the merger.

## Consequences

Repositories permit merge commits in the history of their primary branch, called the default branch. Each authored commit lands with its original identity and message. Cryptographic signatures, Git hooks (scripts that run on Git events), secret scanning and [identity trailers](https://github.com/tbhb-dev/rules/blob/main/docs/reference/commit-identity.md) apply to each authored commit.
