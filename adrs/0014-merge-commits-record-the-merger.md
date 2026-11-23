---
type: Decision Record
title: Merge commits record the merger
description: A Merged-by line in the commit message identifies who merged the pull request.
tags: [merges, provenance, git]
status: stable
decided: "2026-11-11"
---

# Merge commits record the merger

## Context

The merge command can write a commit on a maintainer’s behalf. Commit authorship alone does not always identify who took responsibility for the merge.

This rule attributes the act of merging a pull request and leaves commit authorship separate.

## Decision

The commit that lands the pull request carries a `Merged-by` trailer, a line added to the end of the commit message to identify who merged it. The trailer accompanies the pull request description converted to plain text for the commit message.

## Consequences

The Git history retains the merger’s attribution alongside the pull request description.
