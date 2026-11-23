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

A pull request may be stale, superseded or ready to close without merging. A recorded reason tells the next contributor what happened and where to continue.

## Decision

Agents use the `tbhbdev` pull request close command with a reason and a reference to the relevant work. Closing keeps the branch. When the close guard cannot read the approvals needed to authorize a pull request or issue close, it refuses the request and reports the error.

## Consequences

The close event and the command's confirmation provide evidence of closure. A blocked close can be retried when approvals are readable. Reopen the pull request to undo a close. A reviewed change to the guard can reverse its handling of approval read errors.
