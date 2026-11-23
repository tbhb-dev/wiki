---
type: Decision Record
title: Merge commits record the merger
description: The merge commit identifies the actor responsible for merging the pull request in a Merged-by trailer.
tags: [merges, provenance, git]
status: stable
decided: "2026-11-11"
generated:
  - role: generated
    harness: codex:0.171.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:4f00750704d1bb6e
    at: "2026-11-23T02:44:49Z"
---

# Merge commits record the merger

## Context

The merge command can write a commit on a maintainer’s behalf. Commit authorship alone does not always identify who took responsibility for the merge.

## Decision

The merge commit carries a `Merged-by` trailer that identifies the merger. The trailer accompanies the cleaned pull request body in the commit message.

## Consequences

The Git history retains the merger’s attribution alongside the pull request description.
