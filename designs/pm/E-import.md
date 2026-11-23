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

The import uses GitHub issues and the org Project. One Epic in the central work repository covers the import. Each batch has a Chore under the Epic. The Chore records the batch query and outcome counts. About 28 batches of 25 fit within GitHub's cap of 100 sub-issues per parent.

## Staging and refinement

The imports role maps each source item into a Markdown file in the import run's working directory, outside any repository. It selects the destination type and repository. To check the title, the imports role uses the form for the responsible role. It sets the area labels and Priority. The imports role applies the [issue writing rules](D-issue-writing.md#imports) to the text in each destination section. Defects, Features, Spikes, Tasks, and Chores need a `work-type` value in staging. Work type defaults to `Planned`. When the source describes an unplanned interruption, the imports role uses `Unplanned`.

Under the [routing order](B-routing-and-taxonomy.md#routing-order-for-create), an item about internal security goes to a private repository.

For a Defect, the imports role takes Severity from the source text when it gives a value. Otherwise, Severity defaults to SEV3, the lowest severity. At pickup, the responsible role confirms Severity.

A separate run of the imports role applies the [Definition of Ready](C-refinement.md#readiness-checks) to the staged file. In staging, the refiner records the outcome and its evidence. For any owner flag, the record gives the reason. A `needs-info` record identifies the answering role or specifies that the operator must answer. Follow-up belongs to the responsible role.

Every staged issue must pass `cue vet` against its destination type before creation, including flagged issues. The check covers the required sections, fields, and caps. The kernel also checks the required relations and labels. The import sets Source to `Import` using the destination variant's optional `source` field. For a Problem, that value meets the [Problem Source requirement](../itsm-records/problems.md#fields).

The content check runs the scans and the destination type's writing checks. It runs the same [Vale pass](C-refinement.md#vale-pass) on the rendered body that the kernel runs at issue creation. The pass uses the same organization configuration and the destination variant's short-body threshold.

1. A Vale refusal stops the staged file before the kernel writes it. The imports role reads the findings in staging.
2. The imports role applies only fixes that preserve intent under the [refinement fix rules](C-refinement.md#steps-for-each-item). If a fix would change intent, the run stops the batch and keeps the draft in staging.
3. The imports role renders the revised draft and reruns the staging check. A second Vale refusal for the same draft stops the batch. The draft and findings stay in staging for the pm role. The run log shows counts only.

## Review and creation

The independent reviewer uses the [batch review checklist](F-refinement-runs.md#review-checklist) with the staged files and evidence. For `split`, `drop`, and `needs-info`, the reviewer checks that the flag matches the evidence and gives the responsible role enough information to follow up. The reviewer posts the verdict on the batch Chore.

Once the batch passes review, the kernel repeats the checks. Each issue gets one write. The creation write supplies the vetted body and the destination type. If the staged outcome is `ready` and the issue has no owner flag, that write adds the `ready` refinement label. When a staged outcome calls for owner follow-up, the kernel adds `flag:split`, `flag:drop`, or `flag:needs-info` in that write. An owner flag appears alongside the labels for the issue type.

On the batch Chore, the kernel records the created issue link and the vetted body hash beside its staging result. The sync job uses the [Ready signal](C-refinement.md#status-and-the-ready-signal) to match a staged `ready` result to the created issue.

For an open issue with an owner flag, the sync job clears Status if its refinement record is missing or stale.

On retry, the import checks for a GitHub issue already created from the same source item. It resumes from that issue if it exists. If the run cannot tell whether a prior write succeeded, it stops the batch.

## Order and batches

1. Order by destination type. Initiatives and Epics come first, then Features and Spikes, then Defects and Chores.
2. Within a level, take `Expedite`, then `Standard`, then `Intangible`. Take the oldest first within each Priority.
3. Wait to refine a child until its parent has a staged outcome. A child whose parent has a `drop` outcome gets a new parent or the `flag:drop` label at creation. A required parent link must pass the kernel check before creation.
4. Keep each batch to one target repository and at most 25 items.

## Throttling

GitHub documents a secondary limit of 80 content-creating requests a minute and 500 an hour. Writes in the web interface count toward it. GitHub can change those numbers without notice. No endpoint reports how much of the limit remains.

Import and refinement runs share one budget of at most 200 writes an hour and 20 a minute. One run writes to a target repository at a time. On a 403 or 429 response, a run waits with a delay that doubles each time. The run stops when the budget runs out.

Review sets the pace as well. At most two batches wait for review at once, and the launcher doesn't start another batch until one of them clears review.

## Calibration

Before the first production batch, the imports role stages the 20 calibration items and loads them as their destination types in a test repository. The set includes a done item and a duplicate. It also includes an item too large for one pull request, stale context, a check that says only that someone did the work, and a sentence about the issue itself. The pm role compares the outcomes with the known answers. The pm role tunes the [run brief](F-refinement-runs.md#run-brief) before the import starts.

## Finish line

The finish checks count every staged draft. A creation refusal fails the batch. The import ends when all of the following checks pass.

- Every staged source item has one GitHub issue in its destination type with Source `Import`.
- The batch Chore records each issue's staging outcome.
- Each issue with a `split`, `drop`, or `needs-info` outcome receives its matching `flag:split`, `flag:drop`, or `flag:needs-info` label at creation.
- Each batch Chore has closed after passing review.
- The import Epic's closing comment gives the outcome counts.

After 30 days without an answer, an import with `flag:needs-info` gets one reminder. The pm role may then drop it as not planned. A drop record cites the unanswered question. Items that wait on the operator are exempt.
