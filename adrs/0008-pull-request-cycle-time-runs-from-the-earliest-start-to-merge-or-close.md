---
type: Decision Record
title: Pull request cycle time runs from the earliest start to merge or close
description: A pull request's cycle time starts at the earliest started_at on its identity trailers or its earliest commit time, and ends at merge or at close without merge.
tags: [metrics, cycle-time, pull-requests, trailers]
status: stable
decided: "2026-11-15"
---

# Pull request cycle time runs from the earliest start to merge or close

## Context

Pull request (PR) timing stats and review-round figures need the real time from the first work on a change to its end. Each identity trailer line on a commit carries a `started_at` field in UTC, the start of the agent run behind that line. A PR can hold several runs, Generated-by and Assisted-by alike, and commits that no agent made.

## Decision

A PR's cycle time starts at the earliest of these:

- every `started_at` on the identity trailer lines in the PR's commits or body, Generated-by and Assisted-by alike;
- the commit time of the PR's earliest commit.

It ends at the merge time, or at the close time for a PR closed without merging. Closed PRs count and are reported as a separate state. Work before coding isn't counted.

A start-source label names which start won, `started_at` or `commit`, and every stat and metric carries it.

## Options considered

- Count only Generated-by lines: rejected because assisting runs are part of the work.
- Use the earliest commit time only as a fallback when no `started_at` exists: rejected because a commit with no agent that predates every agent start is still earlier work.

## Consequences

`tbhbdev alpha pr-stats` and the org's GitHub metrics read the same definition. Until the metrics service has a read-only way to list a PR's commits, `tbhbdev alpha pr create` writes the oldest commit trailer's `started_at` into the PR body.

A long-lived session's trailer holds the start of the session, not the start of the work, so a PR from such a session shows a longer cycle time than it took.
