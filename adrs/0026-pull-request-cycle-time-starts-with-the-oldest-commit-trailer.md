---
type: Decision Record
title: Pull request cycle time starts at the earliest recorded time
description: Pull request cycle time runs from the earliest recorded agent start or commit time to merge or unmerged close.
tags: [cycle-time, metrics, pull-requests]
status: stable
decided: "2026-11-22"
---

# Pull request cycle time starts at the earliest recorded time

## Context

Pull request cycle time measures elapsed time from when work began. Work can start before the first commit or before the pull request opens. A change can include work from several agent runs, or sessions with automated contributors. The measure covers merged and closed pull requests and excludes those still open.

## Decision

Cycle time starts at the earliest candidate timestamp. The pull request's earliest commit time is always a candidate. Every `started_at` on the pull request's identity lines is a candidate. Identity lines can appear in commits or in the pull request body. [Commit identity trailers](https://github.com/tbhb-dev/rules/blob/main/docs/reference/commit-identity.md) are attribution lines at the end of commit messages. A `started_at` timestamp records when an agent run began. `Generated-by` lines count for agents that wrote changes. `Assisted-by` lines count for agents that helped. The start-source label names the source of the earliest timestamp, `started_at` or `commit`. Cycle time ends at merge or at closure without merge. Metrics report closure without merge as a separate state.

## Consequences

The metrics integration collects pull request timing data. The command that opens a pull request copies the earliest of the trailer timestamps and the earliest commit time into the description until the integration can list commits with read-only access. A long-running agent session can start before it begins work on a change, which makes the measured cycle time longer.
