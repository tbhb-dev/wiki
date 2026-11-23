---
type: Decision Record
title: Pull request cycle time starts at the earliest recorded start
description: Pull request cycle time measures the time to merge or unmerged closure from the earliest start recorded in its commit identity trailers or, when no trailer records a start, the commit time of its earliest commit.
tags: [cycle-time, metrics, pull-requests]
status: stable
decided: "2026-11-22"
---

# Pull request cycle time starts at the earliest recorded start

## Context

Pull request cycle time measures elapsed time from when work began. Work can start before the first commit or before the pull request opens. A change can include work from several agent runs, or sessions with automated contributors. The measure covers merged and closed pull requests and excludes those still open.

## Decision

Cycle time starts at the earliest `started_at` timestamp in the pull request's [commit identity trailers](https://github.com/tbhb-dev/rules/blob/main/docs/reference/commit-identity.md), which are attribution lines at the end of a commit message. The timestamp records when an agent run began. The calculation includes `Generated-by` for agents that wrote changes and `Assisted-by` for agents that helped. When no trailer carries `started_at`, cycle time starts at the commit time of the pull request's earliest commit. Cycle time ends at merge or at closure without merge. Metrics report closure without merge as a separate state.

## Consequences

The metrics integration collects pull request timing data. The command that opens a pull request copies the earliest trailer timestamp into the description until the integration can list commits with read-only access. A long-running agent session can start before it begins work on a change, which makes the measured cycle time longer.
