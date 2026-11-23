---
type: Decision Record
title: Agents close pull requests with a reason
description: Agents close pull requests through tbhbdev, record the reason and keep the branch available for reopening.
tags: [agents, github, pull-requests]
status: stable
decided: "2026-11-15"
---

# Agents close pull requests with a reason

## Context

A pull request may be stale, superseded or ready to close without merging. A recorded reason tells the next contributor what happened and where to continue. The rule covers pull requests closed by agents, meaning automated contributors, and approval requirements for issue closures, leaving merge procedures outside its scope.

## Decision

Agents use the [tbhbdev pull request close command](https://github.com/tbhb-dev/tools/blob/main/docs/reference/alpha/no-self-approval.md) with a reason and a reference to the relevant work. The command records the reason and submits the close. Closing retains the branch. The close guard must refuse to close a pull request or issue when it cannot read the required approvals, and it must report the error. The command has yet to enforce the guard, which is tracked as follow-up work in the tools repository.

## Consequences

The GitHub event recording the closure and the command's confirmation provide evidence of closure. Once the guard is in place, a blocked close can be retried when approvals are readable. Reopen the pull request to undo a close. A reviewed change to the guard can reverse its handling of approval read errors.
