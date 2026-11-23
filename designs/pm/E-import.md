---
type: Design Appendix
title: Project management tracker import
description: How about 700 issues from the earlier tracker enter through the kernel and pass refinement, in what order, at what write rate, and when the import ends.
tags: [pm, import, refinement, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Project management tracker import

Back to [the design](README.md).

Plan step 7 imports about 700 issues from the earlier tracker. Each one enters through the kernel in an imported variant, which keeps the title and one Context section of the original text.

That variant runs the scans without section or cap rules, and it reports the checks of [D. Issue writing](D-issue-writing.md) as warnings.

While it maps the original text, the import strips each sentence that describes the issue itself, as D describes. It sets Source to `Import` and keeps that value when refinement converts the issue to its real variant. A converted Problem with Source `Import` meets the Source requirement in both this design and the [Problem design](https://github.com/tbhb-dev/wiki/blob/main/designs/itsm-records/problems.md#fields). An item about internal security goes in a private repository, as rule 6 of D requires.

Each imported item then shows Needs refinement, and nobody can start it until refinement under [C. Refinement](C-refinement.md) turns it into a real variant.

The import uses issues and the org Project, without a new tracker. One Epic in the central work repository covers the import, and each batch is a Chore under it with its query and outcome counts. About 28 batches of 25 fit well under GitHub's cap of 100 sub-issues per parent.

## Staging and the Vale pass

Before any write, the import maps each original issue into a Markdown file in the import run's working directory, outside any repository. The content check then runs over each staged file. It runs the scans and the checks of D as warnings. It runs the same [Vale pass](C-refinement.md#vale-pass) on the rendered body that the kernel runs at issue creation, with the same organization configuration and semicolon rule.

The imported variant has no total-word cap, so Vale errors become warnings only for bodies under 65 prose words. Any draft that the creation check would refuse also fails staging. The import posts no finding list, because the run's logs show counts only.

The kernel reruns the same pass at creation and runs it again with the destination variant's threshold when refinement converts the issue.

## Order and batches

1. Order by the type that the import gives each item. Initiatives and Epics come first, then Features and Spikes, then Defects and Chores.
2. Within a level, take `Expedite` before `Standard` before `Intangible`, then the oldest first.
3. Wait to refine a child until its parent has an outcome. A child under a dropped parent gets a new parent or a Drop of its own.
4. Keep each batch to one target repository and at most 25 items.

## Throttling

GitHub documents a secondary limit of 80 content-creating requests a minute and 500 an hour, and writes in the web interface count toward it. GitHub can change those numbers without notice, and no endpoint reports how much of the limit remains. A refinement writes two to five times per item, so 700 items need a few thousand writes.

Refinement runs share one budget of at most 200 writes an hour and 20 a minute, with one run per target repository at a time. That leaves room under the documented limits for the kernel's other writes. On a 403 or 429 response, a run waits with a delay that doubles each time, and it stops when the budget runs out.

Review sets the pace as well. At most two batches wait for review at once, and the launcher doesn't start another batch until one of them clears review.

## Calibration

Before the first real batch, a run refines 20 imported items in a test repository. The set includes known bad cases. Two are a done item and a duplicate.

Others are an item too large for one pull request, stale context, a check that says only that someone did the work, and a sentence about the issue itself. The pm role compares the outcomes with the known answers and tunes the run brief of F before the import starts.

## Finish line

The finish checks count every staged draft. A creation refusal fails the batch. The import ends when every staged draft has passed creation, has a recorded refinement outcome, and the following checks pass:

- Zero imported items show Needs refinement.
- Each Needs info item gives its answerer.
- Each batch Chore has closed after a passing review.
- The import Epic's closing comment gives the outcome counts.

A Needs info item without an answer after 30 days gets one reminder. After that, the pm role may drop it as not planned, citing the unanswered question in the record. Items that wait on the operator are exempt.
