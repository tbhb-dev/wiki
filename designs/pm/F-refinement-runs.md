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

# Project management refinement runs

Back to [the design](README.md).

This appendix gives the parts of refinement that a launching run and a reviewer use directly. The Definition of Ready (DoR), the procedure, and the Vale pass stay in [C. Refinement](C-refinement.md), and item numbers here refer to its DoR. D stands for [D. Issue writing](D-issue-writing.md), and D8 sits in [decisions](decisions.md).

Everything here is new.

## Run brief

The launching run fills the placeholders in angle brackets and pastes the text as the refinement run's prompt.

```text
You are a refinement run for the <pm | sm> role. Refine the items below
against the Definition of Ready in designs/pm/C-refinement.md of the
tbhb-dev/wiki repository, and write each result with `tbhbdev pm refine`.

Items: <issue list, or the query that selects them>
Batch Chore: <owner/repo#number, or none for a single item>
Repository commits to read: <owner/repo@sha for each target repository>
Label configuration: <path@sha of the tools label configuration>
Write budget: <writes per hour and per minute>, shared with other runs

For each item, in the order given:
1. Read the item, its comments, its parent, its children, and its
   blocked-by links.
2. Run `tbhbdev pm refine --check <item>` and read every finding and
   warning.
3. Walk items 1 to 8 of the DoR. For each, write pass, fixed, or fail,
   with evidence: a SHA, a pull request, a path at a SHA, or a command
   with the commit or query it ran against and its result.
4. Choose one outcome: ready, fixed-then-ready, needs-info, split, or drop.
5. Run Vale with the org configuration (.vale.ini in tbhb-dev/rules)
   on the final body and on each comment you will post.
6. Write it with `tbhbdev pm refine <item> --outcome <outcome>`.

Rules:
- Fix wording, stale context, empty fields, and links. Never change the
  item's scope. Ask instead, as needs-info.
- Each outcome and check states a result someone can observe. A check
  that says only that work exists or was done fails item 3.
- Each sentence states one fact. Each list item is a full sentence that
  ends with a period.
- Delete each sentence about the issue itself, its tracking, or its
  links. Sub-issue and dependency links carry those.
- Item 4 needs a repo search and an issue search, both cited.
- A Vale error blocks the write. In a short text, every Vale finding
  is a warning. Report each finding with its line, rule, and level.
- Vale never edits text. Make each fix yourself, under the fix rules.
- Move an internal-security item out of a public repository first,
  by a transfer to its private twin.
- Add no owner line. A Defect's parent is a Problem or an Epic, or none.
- Give the answerer of each question. Ask the operator only about
  operator intent.
- Skip any item that already has a current record.
- Never refine an item whose body you wrote.
- Cite no private repository in a public item.
- On a 403 or 429 response, back off. Stop when the write budget runs
  out, and report where you stopped.

At the end, report the count of each outcome, the items you skipped and
why, and any check you couldn't run.
```

## Review checklist

A second run reviews each batch before its Chore closes. It reads as a reviewer only, from a model family other than the refiner's, and posts its verdict as a comment on the batch Chore. On a pass, a pm or `sm` run closes the Chore, and on a fail, that run removes the ready labels in the batch.

A refinement of one new item has no batch. Its Ready shows at once. Each week, a reviewer samples those records the same way, as if the week's records formed one batch.

- [ ] The reviewer reads every item in a batch of 10 or fewer. In a larger batch, it takes 10 items or a fifth at random, whichever is more, and adds each Drop, each Split, and each fix that edited the outcome or the checks.
- [ ] A sampled record covers items 1 to 8, each with a result and evidence that meets the [evidence rules](C-refinement.md#evidence-rules).
- [ ] The reviewer runs at least one cited command per sampled item again, at the cited commit, and gets the same result.
- [ ] For item 4, the reviewer's own search doesn't turn up done work or a duplicate that the refiner missed.
- [ ] Each outcome and check on a sampled Ready item states an observable result, and none says only that the work exists or that someone did it.
- [ ] A sampled Ready item has no semicolon in its prose, each of its list items ends with a period, and none of its sentences describes the issue itself.
- [ ] The reviewer runs Vale with the org configuration again on each sampled Ready item and its record. The rerun shows the warnings the record lists and, outside a short text, zero errors.
- [ ] Each sampled item sits in the repository the routing order picks. An internal-security item sits in a private repository.
- [ ] No sampled item has an owner field or an owner line. A `role/<role>` label appears only where the work belongs to another role.
- [ ] A sampled Defect has a Problem, an Epic, or no parent.
- [ ] For each fix, the edit history shows that the item's scope didn't change.
- [ ] Each Needs info question gives an answerer, and none asks the operator about something a role could answer.
- [ ] Each Drop cites the change, the duplicate, or the reason that has gone, and the close reason matches it.
- [ ] Each Split child is in the right repository and links to the original. It fits one pull request.

A rubber-stamp Ready leaves signs, and one of them sends the reviewer to read the whole batch:

- Over three in four imported items come out Ready without a fix, though imported text rarely passes items 2 and 8 unedited.
- Two or more records share the same evidence text, or cite a commit without a command.
- A Ready item still matches a pattern of item 2, 3, 7, or 8 when checked again.
- An item 4 pass lacks an issue search.

The batch fails when two or more sampled items fail, or when a full read confirms a rubber stamp. A failed batch gets a new refinement run, never the same one.

## Failure modes

| Failure | Effect | Safeguard |
| --- | --- | --- |
| A refiner marks items Ready without checking | Workers start on unclear or done work | The sampled review and the rubber-stamp signs |
| A fix changes the item's scope | The work no longer matches the author's intent | The fix rules, and the reviewer's look at the edit history |
| The DoR turns into a phase gate | Work waits for detail a worker could find while working | The DoR doesn't require a design or an estimate, exploration splits off as a Spike, and fixes happen in place |
| A check says only that someone did the work | Someone closes the item without a result anyone saw | Item 3, the existence patterns of D, and the review checklist |
| Someone edits a Ready item | The record no longer describes the text | The body hash, which returns the item to Needs refinement |
| Someone adds the ready label by hand | An item looks ready without a record | The sync job ignores a label without a current record |
| A built-in Project workflow writes Status | Status disagrees with the records | The sync job's setup turns off each built-in workflow that sets Status |
| Needs info piles up for the operator | Operator time goes to triage | The rule that the operator answers only intent, and D8 |
| A run stops partway through a batch | Part of a batch has records | Runs skip items with a current record and resume |
| An internal-security item sits in a public repository | Anyone can read about the weakness | The security-topic warning of D, item 5, and a transfer before any other write |
| Vale's document-wide rules flag a short body | An honest short text can't pass | Warnings only below the short threshold |
| A record cites private evidence on a public item | A private detail leaks | The kernel's private-detail scan refuses the write |
| Merged work makes a Ready item moot later | Someone repeats done work | The worker checks item 4 again before opening a pull request |
