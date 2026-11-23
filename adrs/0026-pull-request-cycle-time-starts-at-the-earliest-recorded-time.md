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

Cycle time starts at the earliest candidate timestamp. Commit time means committer date, so rebased commits count from their rebase time. The earliest committer date among the pull request's commits is always a candidate. A `started_at` timestamp records when an agent run began. Only `Generated-by` and `Assisted-by` [commit identity trailers](https://github.com/tbhb-dev/rules/blob/main/docs/reference/commit-identity.md) in the pull request's commits or body can supply `started_at` candidates. Label the start `started_at` when the earliest trailer candidate precedes or equals the earliest committer date. Otherwise use `commit`. Cycle time ends at merge or at closure without merge. Metrics report closure without merge as a separate state.

## Consequences

The metrics integration collects pull request timing data. The command that opens a pull request copies the earlier of the earliest `started_at` and the earliest commit time into the description. The copied value keeps the label of its source. This step remains necessary until the integration can list commits with read-only access. Work on a change can begin partway through an agent session. In that case, the session start makes the measured cycle time longer.
