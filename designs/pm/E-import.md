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

About 700 issues enter the backlog from the earlier tracker under the [import plan](README.md#plan). In staging, the imports role refines the files before the kernel creates each issue in its destination type.

GitHub issues go on the target repository's org Project. One Epic in the central work repository or its private twin covers the import. Give each batch a Chore under the Epic. Record the batch query and outcome counts on the Chore. About 28 batches of 25 fit within GitHub's cap of 100 sub-issues per parent.

## Staging and refinement

Store each item's original text in a Markdown file in the run's working directory outside any repository. The imports role chooses the destination type and repository. For fields and labels, use the [shared taxonomy](B-routing-and-taxonomy.md#one-home-per-fact). Follow the [shared writing rules](../issue-refinement/B-definition-of-ready.md#writing-rules) in each section.

Classify each item's Source from its origin notes and tracker text. Record a one-line reason under `origin:` in the working notes. Keep that key and those notes in staging. Never send them to GitHub. Put each item with an unclear origin on a short list for the owner of its area. Leave Source empty until the owner answers.

For a review finding, cite the pull request or finding in Context.

Under the [routing order](B-routing-and-taxonomy.md#routing-order-for-create), an item about internal security goes to a private repository.

A separate run of the imports role applies the [Definition of Ready](../issue-refinement/B-definition-of-ready.md#readiness-checks) to the staged file.

Thirty-two staged titles start with a process verb from the [shared title rules](../issue-refinement/B-definition-of-ready.md#titles-and-process-verbs). During refinement, rewrite those titles. Run the title lint from refinement onward when the later pm schema change makes it available. Use the imported variant to preserve the original title text.

Record each outcome with its evidence in staging. A `needs-info` record identifies the answering role or specifies that the operator must answer. Give follow-up to the responsible role.

Resolve splits in staging by preparing replacement items for their own readiness checks. For a dropped item, leave the draft and its reason in staging.

Each item due for creation must pass `cue vet` against its destination type, including items that need information.

The schema checks the required sections, fields, and caps. The kernel also checks the required relations and labels.

The content check runs the scans and the destination type's writing checks. It also runs the [Vale pass](../issue-refinement/A-refinement-procedure.md#vale-pass) on the rendered body.

The pass uses the same organization configuration and short-body threshold as issue creation.

1. A Vale refusal stops the staged file before the kernel writes it. The imports role reads the findings in staging.
2. The imports role applies only fixes that preserve intent under the [refinement fix rules](../issue-refinement/A-refinement-procedure.md#steps-for-each-item). If a fix would change intent, the run stops the batch and keeps the draft in staging.
3. The imports role renders the revised draft and reruns the staging check. A second Vale refusal for the same draft stops the batch. The draft and findings stay in staging for the pm role. The run log shows counts only.

## Review and creation

An independent reviewer checks the staged files and evidence against the [batch review checklist](F-refinement-runs.md#review-checklist). For `split`, `drop`, and `needs-info`, check whether the outcome matches the evidence. Post the verdict on the batch Chore.

After a passing review, the kernel repeats the checks. Each approved item goes through the [issue write path](README.md#writing-an-issue) once.

Leave dropped items and the original item of a split in staging. For a split, create only the approved replacements.

On the batch Chore, the kernel records the created issue link and the vetted body hash beside its staging result. The sync job matches each created issue to its staging result and sets Status using the [shared Status transitions](../issue-refinement/D-status-transitions.md#transition-order).

On retry, the import looks for a GitHub issue from the same tracker item. Resume from that issue if it exists. Stop the batch if the status of a prior write remains unclear.

## Order and batches

1. Order by destination type. Initiatives and Epics come first, then Features and Spikes, then Defects and Chores.
2. Within a level, follow the shared Priority order. Take the oldest first within each Priority.
3. Wait to refine a child until its parent has a staged outcome. If the parent has a `drop` outcome, give the child a new parent or drop the child in staging. A required parent link must pass the kernel check before creation.
4. Keep each batch to one target repository and at most 25 items.

## Throttling

GitHub documents a secondary limit of 80 content-creating requests a minute and 500 an hour. Writes in the web interface count toward it. GitHub can change those numbers without notice. No endpoint reports how much of the limit remains.

Tracker import and refinement runs share one budget of at most 200 writes an hour and 20 a minute. One run writes to a target repository at a time. On a 403 or 429 response, a run waits with a delay that doubles each time. The run stops when the budget runs out.

Review sets the pace as well. At most two batches wait for review at once, and the launcher doesn't start another batch until one of them clears review.

## Calibration

Before the first production batch, the imports role refines the 20 calibration items in staging. For each approved item, the kernel creates an issue in its destination type in the calibration repository. It adds the issue to `work-calibration`. Include a done item and a duplicate. Other cases cover an item too large for one pull request, stale context, a check that says only that someone did the work, and a sentence about the issue itself. Compare the outcomes with the known answers. Then the pm role tunes the [run brief](F-refinement-runs.md#run-brief) before the import starts.

## Finish line

Count each staged draft before closing the import. A creation refusal fails the batch. Finish when these checks pass.

- Staging shows an outcome for each item.
- Each approval for creation has one GitHub issue in the destination type.
- Find a one-line reason in the working notes for each Source classification. Check the list for each area owner for items with unclear origins.
- The batch Chore records the staging outcome for each created issue.
- Each open imported issue with a current `needs-info` result shows Status `Needs info`.
- Each batch Chore has closed after passing review.
- The import Epic's closing comment gives the outcome counts.

After 30 days without an answer, an imported issue with Status `Needs info` gets one reminder. The pm role may then close it as not planned through ordinary refinement. The record cites the unanswered question. Items that wait on the operator are exempt.
