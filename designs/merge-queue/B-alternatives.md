---
type: Design Appendix
title: In-house merge queue alternatives
description: The options weighed against building the merge queue as Temporal workflows, and why each lost.
tags: [merge-queue, alternatives, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# In-house merge queue alternatives

Back to [the design](README.md).

## Compared

| Option | Private repositories | Cost | Smarter checks and metrics | Fits merge commits, signatures, no bypass | Verdict |
| --- | --- | --- | --- | --- | --- |
| A. Temporal workflows and a Go worker, this design | Yes | Host time only | Yes, built in | Yes, through E1 and the merge endpoint | Recommended |
| B. A plain Go service with its own SQLite state | Yes | Host time only | Yes | Yes | Rejected, the operator chose Temporal |
| C. A serial lock inside `tbhbdev alpha merge` | Yes | None | No | Yes | A stopgap at most |
| D. Upgrade to GitHub Enterprise Cloud | Yes | Money, per seat per month (unverified: the current price) | No, the same queue as today | Yes | Rejected, spends money |
| E. A hosted queue product | Yes | Paid plans for private repositories (unverified: each vendor's current pricing) | Some | Many push a staging branch to main, which needs a bypass actor | Rejected, spends money and needs a bypass |

## B. A plain service

A single Go process with its own SQLite tables for queue order, groups, and attempts would reach a first merge sooner, since it needs no Temporal server. It would also need its own crash recovery, timers, and history view, which Temporal gives for free.

The operator decided that Temporal orchestrates the review and merge queue (existing: the operator's decision of 2026-11-22). This design follows that decision. It still keeps the queue's logic in plain packages, and it starts on the single-binary Temporal server so the missing production server doesn't hold up the first cut.

## C. A serial lock in the merge command

`tbhbdev alpha merge` would take a lock per repository, build the merge locally, wait for CI on a pushed candidate, and merge. It needs no server.

It runs inside whichever agent called it, so a stopped agent strands the lock, and nobody sees the queue in one place. It can't batch, can't build candidates at once, and keeps no flake or timing history. Plan step 2 makes it unnecessary.

## D. Enterprise Cloud

It would turn GitHub's queue on in private repositories. It costs money every month, which the Laws forbid committing to, and it adds none of the missing parts. The operator has already declined the upgrade (existing: the operator's decision of 2026-11-24).

## E. Hosted products and open-source bots

Hosted merge queues exist with free tiers for open source and paid plans for private code (unverified: the vendors' current terms). Older open-source bots of the bors family test a staging branch and then push it to main, which needs push rights that the no-bypass rule rules out. None of them knows Bazel's cache or tbhb-dev's message and approval rules, and a hosted one would hold write access to every private repository.

## The earlier review and merge queues design

The automated review and merge queues design specified a merge queue inside the `tbhbdev alpha loop` worker in its appendix G (existing: tbhb-dev/wiki-private `designs/pr-review-merge-queues/`). The Temporal decision already makes the newer workflow design win where they conflict. This design replaces that merge half, and its review half stays as written.

| Earlier design | This design | Why |
| --- | --- | --- |
| Squash merges, with a tree check against a squash candidate | Merge commits | [ADR 0020](../../adrs/0020-pull-requests-land-through-merge-commits.md) came later and requires merge commits (D4) |
| One merge in flight per repository, no batching | Serial first, then groups and concurrency | Today's GitHub queues already batch up to 5, and serial retests cost too much at tools' pass times |
| Candidates on `queue/<N>/...` | Candidates on `merge-queue/main/<entry>` | One candidate per branch, and the name says what it is |
| Retest only when the base moved | Always test the projected main, with the proven-tree skip covering the unchanged case | The same saving, with one rule instead of two |
| Conflicts rebuilt by cherry-pick into a replacement pull request | The queue removes a conflicted entry, and the author updates the branch | [ADR 0021](../../adrs/0021-authors-may-update-and-curate-pull-request-branches.md) now lets authors merge or rebase main into their branch |
| The live mode of `tbhbdev alpha loop` | Temporal workflows with a worker in its own repository | The operator's Temporal decision, and the request for a project of its own |

The earlier design's decided items that still hold carry over as D5 to D8.
