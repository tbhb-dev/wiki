---
type: Design Appendix
title: Project management tracker import
description: How the imports role refines about 700 staged tracker items and creates each issue in its destination type.
tags: [pm, import, refinement, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Project management tracker import

Back to [the design](README.md).

The [import plan](README.md#plan) brings in about 700 issues from the earlier tracker. The imports role refines the staged files before the kernel creates each issue in its destination type.

The import uses GitHub issues and the target repository's org Project. One Epic in the central work repository covers the import. Each batch has a Chore under the Epic. The Chore records the batch query and outcome counts. About 28 batches of 25 fit within GitHub's cap of 100 sub-issues per parent.

## Staging and refinement

The imports role maps each source item into a Markdown file in the import run's working directory, outside any repository. It keeps the original text in staging. The `origin:` working key records where that text came from and stays in staging. The imports role selects the destination type and repository. It sets the labels from [B. Routing and taxonomy](B-routing-and-taxonomy.md#one-home-per-fact). The imports role applies the [issue writing rules](D-issue-writing.md#imports) to the text in each destination section. Defects, Features, Spikes, Tasks, and Chores need a `work-type` value in staging. Work type defaults to `Planned`. When the source describes an unplanned interruption, the imports role uses `Unplanned`.

Imported items default to Priority `Standard`. The imports role sets Priority to `Intangible` for an item with no cost of delay today. Examples include a Chore from a review, a mutation survivor, cosmetic or documentation cleanup, and an improvement nothing waits on. Deferred security work stays `Standard`. No imported item gets `Expedite`.

Under the [routing order](B-routing-and-taxonomy.md#routing-order-for-create), an item about internal security goes to a private repository.

For a Defect, the imports role takes Severity from the source text when it gives a value. Otherwise, Severity defaults to SEV3, the lowest severity. At pickup, the responsible role confirms Severity.

A separate run of the imports role applies the [Definition of Ready](C-refinement.md#readiness-checks) to the staged file. Thirty-two staged titles start with a process verb from [D. Issue writing](D-issue-writing.md#the-rules). The refinement pass rewrites those titles. The title lint applies from refinement onward. The imported variant preserves the original title text.

The refiner records each outcome and its evidence in staging. A `needs-info` record identifies the answering role or specifies that the operator must answer. The responsible role handles follow-up. The refiner resolves splits in staging by preparing replacement items for their own readiness checks. A dropped item stays in staging with its reason.

Each item due for creation must pass `cue vet` against its destination type, including items that need information. The check covers the required sections, fields, and caps. The kernel also checks the required relations and labels. Each destination variant accepts an optional `source` entry in `fields`. The kernel sets the Source org issue field to `Import` on each issue it creates. For a Problem, that value meets the [Problem Source requirement](../itsm-records/problems.md#fields).

The content check runs the scans and the destination type's writing checks. It runs the same [Vale pass](C-refinement.md#vale-pass) on the rendered body that the kernel runs at issue creation. The pass uses the same organization configuration and the destination variant's short-body threshold.

1. A Vale refusal stops the staged file before the kernel writes it. The imports role reads the findings in staging.
2. The imports role applies only fixes that preserve intent under the [refinement fix rules](C-refinement.md#steps-for-each-item). If a fix would change intent, the run stops the batch and keeps the draft in staging.
3. The imports role renders the revised draft and reruns the staging check. A second Vale refusal for the same draft stops the batch. The draft and findings stay in staging for the pm role. The run log shows counts only.

## Review and creation

The independent reviewer uses the [batch review checklist](F-refinement-runs.md#review-checklist) with the staged files and evidence. For `split`, `drop`, and `needs-info`, the reviewer checks that the outcome matches the evidence. The reviewer posts the verdict on the batch Chore.

Once the batch passes review, the kernel repeats the checks. It creates each approved item once with the vetted body, destination type, and labels. It writes the org issue fields on the issue. The original item from a split and every dropped item stay in staging. The kernel creates only the approved replacement items from a split.

On the batch Chore, the kernel records the created issue link and the vetted body hash beside its staging result. The sync job matches each created issue to its staging result and sets Status using the [Status table in C. Refinement](C-refinement.md#status-and-the-ready-signal). A current `needs-info` result gives the issue Status `Needs info`.

On retry, the import checks for a GitHub issue already created from the same source item. It resumes from that issue if it exists. If the run cannot tell whether a prior write succeeded, it stops the batch.

## Order and batches

1. Order by destination type. Initiatives and Epics come first, then Features and Spikes, then Defects and Chores.
2. Within a level, follow the Priority option order in [B. Routing and taxonomy](B-routing-and-taxonomy.md#one-home-per-fact). Take the oldest first within each Priority.
3. Wait to refine a child until its parent has a staged outcome. If the parent has a `drop` outcome, give the child a new parent or drop the child in staging. A required parent link must pass the kernel check before creation.
4. Keep each batch to one target repository and at most 25 items.

## Throttling

GitHub documents a secondary limit of 80 content-creating requests a minute and 500 an hour. Writes in the web interface count toward it. GitHub can change those numbers without notice. No endpoint reports how much of the limit remains.

Import and refinement runs share one budget of at most 200 writes an hour and 20 a minute. One run writes to a target repository at a time. On a 403 or 429 response, a run waits with a delay that doubles each time. The run stops when the budget runs out.

Review sets the pace as well. At most two batches wait for review at once, and the launcher doesn't start another batch until one of them clears review.

## Calibration

Before the first production batch, the imports role refines the 20 calibration items in staging. The kernel creates the approved items in the calibration repository in their destination types and adds them to `work-calibration`. The set includes a done item and a duplicate. It also includes an item too large for one pull request, stale context, a check that says only that someone did the work, and a sentence about the issue itself. The pm role compares the outcomes with the known answers. The pm role tunes the [run brief](F-refinement-runs.md#run-brief) before the import starts.

## Finish line

The finish checks count every staged draft. A creation refusal fails the batch. The import ends when all of the following checks pass.

- Every staged item has an outcome in staging.
- Every item approved for creation has one GitHub issue in its destination type with Source `Import`.
- The batch Chore records the staging outcome for each created issue.
- Each open imported issue with a current `needs-info` result shows Status `Needs info`.
- Each batch Chore has closed after passing review.
- The import Epic's closing comment gives the outcome counts.

After 30 days without an answer, an imported issue with Status `Needs info` gets one reminder. The pm role may then close it as not planned through ordinary refinement. The record cites the unanswered question. Items that wait on the operator are exempt.
