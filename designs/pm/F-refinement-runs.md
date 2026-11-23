---
type: Design Appendix
title: Project management refinement runs
description: The brief a refinement run receives, the checklist that reviews each batch, and the failure modes that the review and the procedure guard against.
tags: [pm, refinement, review, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Refinement runs for pm

Back to [the design](README.md).

This appendix gives the pm batch brief, review checklist, and failure modes. Its check numbers refer to the [shared Definition of Ready](../issue-refinement/B-definition-of-ready.md#readiness-checks). Use the [shared procedure](../issue-refinement/A-refinement-procedure.md) to run each check and write the record.

## Run brief

The launching run fills the placeholders in angle brackets and pastes the text as the prompt for the refinement run.

```text
You are a refinement run for the <pm | sm> role. Refine the items below
against the Definition of Ready in
designs/issue-refinement/B-definition-of-ready.md in tbhb-dev/wiki.
Use the procedure in designs/issue-refinement/A-refinement-procedure.md.

Items: <issue list, or the query that selects them>
Batch Chore: <owner/repo#number, or none for a single item>
Repository commits to read: <owner/repo@sha for each target repository>
Label and field configuration: <path@sha for the global and repository rules>
Write budget: <writes per hour and per minute>, shared with other runs

Follow the shared procedure for each item in order, including each staged
file. Apply the title rules under readiness check 11, the hedge check by
type under check 14, and the no-procedure rule under check 15. Use the
service checks for the applicable record type.

For pm batches:
- Use the batch Chore to record review results.
- For import, use a different run from the one that mapped the staged body.
- Follow pm decision D8 for questions of operator intent within a batch.
- On a 403 or 429 response, back off. Stop when the write budget runs
  out, and report where you stopped.

At the end, report the count of each outcome, the items you skipped and
why, and any check you couldn't run.
```

## Review checklist

A batch groups up to 25 items from one target repository under a batch Chore. An independent reviewer from a different model family than the refiner posts a pass or fail verdict on the Chore. In ordinary refinement, a pm or `sm` run closes the Chore after a pass. The [sync job](../issue-refinement/D-status-transitions.md#transition-order) uses the verdict.

A refinement of one new item has no batch. Its Ready shows at once. Each week, a reviewer samples those records the same way, as if the week's records formed one batch.

For the tracker import, the reviewer samples staged files and their evidence before the kernel creates issues. See [E. Tracker import](E-import.md#review-and-creation).

- [ ] The reviewer reads every item in a batch of 10 or fewer. In a larger batch, it takes 10 items or a fifth at random, whichever is more, and adds each Drop, each Split, and each fix that edited the outcome or the checks.
- [ ] A sampled record covers each applicable readiness check with a result and evidence that meets the [evidence rules](../issue-refinement/A-refinement-procedure.md#evidence-rules).
- [ ] The reviewer runs at least one cited command per sampled item again, at the cited commit, and gets the same result.
- [ ] For check 4, the reviewer repeats the searches in the [Definition of Ready](../issue-refinement/B-definition-of-ready.md#readiness-checks) and checks for completed work or duplicates that the refiner missed.
- [ ] Each sampled item meets the [shared title rules](../issue-refinement/B-definition-of-ready.md#titles-and-process-verbs), the [hedge check by type](../issue-refinement/B-definition-of-ready.md#unresolved-hedges), and the [no-procedure rule](../issue-refinement/B-definition-of-ready.md#no-procedure).
- [ ] The reviewer repeats the [Vale pass](../issue-refinement/A-refinement-procedure.md#vale-pass) on each sampled Ready item and its record. The rerun agrees with the record.
- [ ] The applicable [readiness checks](../issue-refinement/B-definition-of-ready.md#readiness-checks) pass for the sampled body and its live metadata and relations.
- [ ] For each fix, the edit history shows that the item's scope didn't change.
- [ ] Each Needs info question gives an answerer, and none asks the operator about something a role could answer.
- [ ] Each `drop` cites the change, the duplicate, or the reason that has gone. For ordinary refinement, the close reason matches the record. For the import, the dropped item stays in staging.
- [ ] Each `split` child fits one pull request and goes to the right repository. For ordinary refinement, the child links to the original. For the import, the kernel creates only approved replacements.

A rubber-stamp Ready leaves signs, and one of them sends the reviewer to read the whole batch:

- More than three quarters of the drafts show a passing result for each readiness check without a fix.
- Two or more records share the same evidence text, or cite a commit without a command.
- A Ready item fails check 2, 3, 7, 8, 11, 14, or 15 on review.
- An item 4 pass lacks an issue search.

The reviewer checks that every `ready` result records a pass or fix for each applicable check. In ordinary refinement, the reviewer also checks that the record is current. The reviewer checks the sample's evidence and inspects the whole batch if there are signs of a rubber stamp. One failed check on any reviewed item fails the batch. A failed batch gets a new refinement run.

## Failure modes

| Failure | Effect | Safeguard |
| --- | --- | --- |
| A refiner marks items Ready without checking | Workers start on unclear or done work | The sampled review and the rubber-stamp signs |
| A fix changes the item's scope | The work no longer matches the author's intent | The fix rules, and the reviewer's look at the edit history |
| The DoR turns into a phase gate | Work waits for detail a worker could find while working | The DoR doesn't require a design or an estimate, exploration splits off as a Spike, and fixes happen in place |
| A check says only that someone did the work | Someone closes the item without a result anyone saw | [Readiness check 3](../issue-refinement/B-definition-of-ready.md#readiness-checks) and the review checklist |
| Someone edits a Ready item or ticks a check | The record no longer describes the body | A change to the body hash makes the record stale. The sync job sets Status to `Needs refinement`. |
| Someone sets Status to `Ready` by hand | An item looks ready without a record | The sync job derives Status from the issue state and the refinement record |
| An assignee or open pull request appears before refinement | An item could appear to be in progress before it passes the gate | In progress requires the item's own current `ready` record, and any batch it belongs to must pass review |
| A built-in Project workflow writes Status | Status disagrees with the records | The sync job's setup turns off each built-in workflow that sets Status |
| Needs info piles up for the operator | Operator time goes to triage | The rule that the operator answers only intent, and D8 |
| Someone answers a Needs info question in a comment. | A comment alone leaves the item at Needs info. | The [question comment](../issue-refinement/A-refinement-procedure.md#outcome-writes) gives the steps to return the item to the queue. The next refinement record cites the answer. |
| A run stops partway through a batch | Part of a batch has records | Runs resume from current records and retry any pending drop close. |
| An internal-security item sits in a public repository | Anyone can read about the weakness | The [internal security procedure](../issue-refinement/A-refinement-procedure.md#internal-security-topics) |
| Vale's document-wide rules flag a short body | An honest short text can't pass | Warnings only below the short threshold |
| A record cites private evidence on a public item | A private detail leaks | The kernel's private-detail scan refuses the write |
| Merged work makes a Ready item moot later | Someone repeats done work | The worker checks item 4 again before opening a pull request |
