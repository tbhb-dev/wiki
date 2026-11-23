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

Squash merges replace the parent branch's commits. For a pull request based on that branch, this can make the same work appear again in the diff and require the author to rebuild the branch.

## Decision

Repositories enable only merge commits for pull request merges. The merge commit subject uses the pull request title with its number, and the body uses the cleaned pull request description. Repository settings back up the merge command's message rules.

See [Pull requests in one repository can be reviewed in parallel](0006-pull-requests-in-one-repository-can-be-reviewed-in-parallel.md) for stacked pull requests. See [Merge commits record the merger](0014-merge-commits-record-the-merger.md) for the `Merged-by` trailer.

## Consequences

Repositories permit merge commits in the history of their default branch. Each authored commit lands with its original identity and message, so commit signatures, hooks, secret scanning and identity trailers apply to each authored commit.
