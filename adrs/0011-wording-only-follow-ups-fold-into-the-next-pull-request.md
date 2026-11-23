---
type: Decision Record
title: Wording-only follow-ups fold into the next pull request
description: A follow-up that changes only wording rides in the next pull request the repository needs, rather than opening alone.
tags: [pull-requests, review, workflow]
status: stable
decided: "2026-11-22"
---

# Wording-only follow-ups fold into the next pull request

## Context

Every pull request (PR) costs CI, a merge and, outside the prose-only repositories, at least one review run. A PR that only rewords text buys little for that cost.

## Decision

A follow-up that changes only wording doesn't get a PR of its own. It goes into the next PR the repository needs anyway.

## Consequences

A wording fix waits until the repository's next PR. Fewer PRs reach the review runs. Undo: open wording-only follow-ups as their own PRs again.
