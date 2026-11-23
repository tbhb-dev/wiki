---
type: Decision Record
title: Automated review can approve delegated permission increases
description: Automated review may approve permission increases for narrow standing allow rules under deploy policy.
tags: [approvals, permissions, deployment]
status: stable
decided: "2026-11-12"
---

# Automated review can approve delegated permission increases

## Context

Deploy policy sets the review rules for changes to agent permissions. Its classes group changes by the authority they need. Standing authorization is ongoing permission to approve qualifying increases.

Class R covers narrow standing allow rules. Each rule has a review date at most 90 days out and no duration limit of its own. Read-only work on another agent host may qualify when it stays within the account’s own clones and worktrees, uses existing destinations, and excludes secrets, other users’ homes, and any widening of privileged access. Wider increases and new or changed decision records fall under class D.

## Decision

The standing authorization follows the deploy policy’s classification of the change. It covers automated approval of class R permission increases. A class D increase requires a decision record already on the default branch at the pull request’s base commit, the commit the proposal builds on.

## Consequences

A class R increase can proceed through normal review without a separate operator request. A recorded decision can revoke the standing authorization.
