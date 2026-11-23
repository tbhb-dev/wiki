---
type: Decision Record
title: Pull requests in one repository can be reviewed in parallel
description: Pull requests can enter review as capacity permits, including when one builds on another.
tags: [reviews, concurrency, pull-requests]
status: stable
decided: "2026-11-22"
---

# Pull requests in one repository can be reviewed in parallel

## Context

A single review slot per repository can idle reviewers. It can also delay independent changes. A stack is a set of pull requests in which a child builds on a parent’s branch. A stack lets authors review a dependent change separately from the work it builds on.

## Decision

Several pull requests in one repository may be open and in review at the same time. Available review capacity and limits on the launcher, the tool that starts review runs, determine how many runs start. Concurrency governs when pull requests can be reviewed and leaves each pull request subject to its own review. Review each stacked pull request against its own base, the branch it targets.

Merge commits preserve the parent’s commit identifiers. After the parent merges, change the child’s base to the repository’s default branch. The child’s review remains valid if its head, the latest commit on its source branch, and its diff, the set of changes under review, stay the same.

## Consequences

The owners responsible for open pull requests can begin their next changes before the current ones merge. A change to the child’s own diff needs review. Rewriting its head calls for a verdict on the new head.
