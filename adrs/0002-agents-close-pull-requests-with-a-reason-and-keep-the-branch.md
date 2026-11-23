---
type: Decision Record
title: Agents close pull requests with a reason and keep the branch, and the close guard fails closed
description: Agents close a pull request with tbhbdev pr close and a reason, the branch stays, and the guard on gh closes blocks when it can't read approvals.
tags: [github, pull-requests, guards]
status: stable
decided: "2026-11-15"
---

# Agents close pull requests with a reason and keep the branch, and the close guard fails closed

## Context

Stalled pull requests (PRs) pile up when nobody closes them, and a close done with `gh pr close` leaves no reason behind and goes around the agent commands. The guard that intercepts `gh pr close` and `gh issue close` exists to stop closes nobody approved, so it reads approvals before it lets one through. That lookup can fail.

## Decision

Agents close a PR with `tbhbdev alpha pr close N --reason "<reason and pointer>"`, never `gh pr close`. The reason says why the PR closed and points to whatever replaces it. The head branch is kept. The evidence of a close is the event relay's closed event and the command's own "Closed pull request" line.

When the close guard can't read approvals, because the lookup raises an error, it blocks the close with a message naming the error. It never lets the close through on a crash.

## Options considered

- Let the guard allow the close when the approval lookup fails: rejected because failing open removes the guard exactly when its input can't be trusted.

## Consequences

A close is undone by reopening the PR, since its branch is still there. A blocked close costs one retry once the approval lookup works again. Issues close the same way, through `tbhbdev alpha issue close` with a reason.

The commands move out of the `alpha` namespace with every other `tbhbdev` command when that namespace is removed. Undo for the fail-closed path: a later PR switches the crash path to allow, with this record superseded.
