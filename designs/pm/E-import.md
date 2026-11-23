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

The imports role maps each source item into a Markdown file in the import run's working directory, outside any repository. The role selects the destination type and repository. It checks the title against the form for the responsible role. It sets the area labels and Priority. It puts the text in the destination type's sections and applies the [issue writing rules](D-issue-writing.md#imports). The [routing order](B-routing-and-taxonomy.md#routing-order-for-create) sends an item about internal security to a private repository.

The refiner applies the [Definition of Ready](C-refinement.md#readiness-checks) to the staged file. The refiner records the outcome and its evidence in staging. If the outcome is `split`, `drop`, or `needs-info`, the imports role prepares the issue in its destination type with a flag for the responsible role. The batch record gives the reason for the flag. For `needs-info`, it also names the role that can answer or identifies the operator. The import leaves the follow-up to the responsible role.

Every staged issue must pass `cue vet` against its destination type before creation, including flagged issues. The check covers the required sections, fields, and caps. The kernel also checks the required relations and labels. The import sets Source to `Import` using the destination variant's optional `source` field. For a Problem, that value meets the [Problem Source requirement](../itsm-records/problems.md#fields).

The content check runs the scans and the destination type's writing checks. It runs the same [Vale pass](C-refinement.md#vale-pass) on the rendered body that the kernel runs at issue creation. The pass uses the same organization configuration and the destination variant's short-body threshold.

1. A Vale refusal stops the staged file before the kernel writes it. The imports role reads the findings in staging.
2. The imports role applies only fixes that preserve intent under the [refinement fix rules](C-refinement.md#steps-for-each-item). If a fix would change intent, the run stops the batch and keeps the draft in staging.
3. The imports role renders the revised draft and reruns the staging check. A second Vale refusal for the same draft stops the batch. The draft and findings stay in staging for the pm role. The run log shows counts only.

## Review and creation

The independent reviewer uses the [batch review checklist](F-refinement-runs.md#review-checklist) with the staged files and evidence. For `split`, `drop`, and `needs-info`, the reviewer checks that the flag matches the evidence and gives the responsible role enough information to follow up. The reviewer posts the verdict on the batch Chore. The kernel creates the issues after the batch passes review.

The kernel repeats the creation checks. The import writes each issue once with its vetted body and destination type. The same write adds the `ready` refinement label or a label that flags the staged outcome for the responsible role. The kernel records the created issue link and the vetted body hash beside its staging result on the batch Chore. The sync job uses a current `ready` record from that Chore after the batch passes review. An owner flag leaves the issue outside Ready until the owner resolves it through ordinary refinement.

On retry, the import checks for a GitHub issue already created from the same source item. It resumes from that issue if it exists. If the run cannot tell whether a prior write succeeded, it stops the batch.

## Order and batches

1. Order by destination type. Initiatives and Epics come first, then Features and Spikes, then Defects and Chores.
2. Within a level, take `Expedite`, then `Standard`, then `Intangible`. Take the oldest first within each Priority.
3. Wait to refine a child until its parent has a staged outcome. A child under a parent flagged `drop` gets a new parent or a `drop` flag of its own. A required parent link must pass the kernel check before creation.
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
- Each issue with a `split`, `drop`, or `needs-info` outcome has its owner flag.
- Each batch Chore has closed after passing review.
- The import Epic's closing comment gives the outcome counts.

An item parked in Needs info during owner follow-up gets one reminder after 30 days without an answer. After that, the pm role may drop it as not planned and cite the unanswered question in the record. Items that wait on the operator are exempt.
