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

This appendix gives the parts of refinement that a launching run and a reviewer use directly. The Definition of Ready (DoR), the procedure, and the Vale pass stay in [C. Refinement](C-refinement.md), and item numbers here refer to its DoR. D stands for [D. Issue writing](D-issue-writing.md), and D8 sits in [decisions](decisions.md).

Everything here is new.

## Run brief

The launching run fills the placeholders in angle brackets and pastes the text as the prompt for the refinement run.

```text
You are a refinement run for the <pm | sm> role. Refine the items below
against the Definition of Ready in designs/pm/C-refinement.md of the
tbhb-dev/wiki repository, and write each result with `tbhbdev pm refine`.

Items: <issue list, or the query that selects them>
Batch Chore: <owner/repo#number, or none for a single item>
Repository commits to read: <owner/repo@sha for each target repository>
Label and field configuration: <path@sha for the global and repository rules>
Write budget: <writes per hour and per minute>, shared with other runs

For each item, including every staged file, in the order given:
1. Read the item, its comments, its parent, its children, and its
   blocked-by links.
2. Run `tbhbdev pm refine --check <item>` and read every finding and
   warning.
3. Walk the applicable items of the DoR. For each, write pass, fixed, or fail,
   with evidence: a SHA, a pull request, a path at a SHA, or a command
   with the commit or query it ran against and its result.
4. Choose one outcome: ready, needs-info, split, or drop. After a fix,
   record ready if every applicable check passes.
5. Run Vale with the org configuration (.vale.ini in tbhb-dev/rules)
   on the final body and on each comment you will post.
6. Write it with `tbhbdev pm refine <item> --outcome <outcome>`.

Rules:
- Fix wording, stale context, empty fields, and links. Never change the
  item's scope. Ask instead, as needs-info.
- Context omits status claims that age. Look for "still", "as of",
  "currently", "confirmed on <date>", and "checked on <date>". Remove
  claims that use an ISO date or HH:MMZ time to say when a state was true.
  Keep fixed facts such as a config date, a version, or a decision date
  with its link.
- Each outcome and acceptance check states a result a reader can see.
  A check that says only that work exists or that someone did it fails
  item 3. Change "Ops rotated the token" to "The setup notes carry the
  new token."
- A Reproduction section gives steps a reader can follow. Use the exact
  placeholder "Find a reproduction and add it here." only for an issue at
  Needs info or a staged item with a needs-info outcome.
- Apply rules 7 through 9 in designs/pm/D-issue-writing.md of tbhb-dev/wiki.
- Each sentence states one fact. Each list item is a full sentence that
  ends with a period.
- Delete each sentence about the issue itself, its tracking, or its
  links. Sub-issue and dependency links carry those.
- Item 4 needs a repo search and an issue search, both cited. For a command
  Request, cite only the search for an open duplicate Request.
- For an Incident, Problem, or Change, apply the common readiness checks
  and use its template and schema for the type-specific checks.
- A Vale error blocks the write. In a short text, every Vale finding
  is a warning. Report each finding with its line, rule, and level.
- Vale never edits text. Make each fix yourself, under the fix rules.
- Move an internal-security item out of a public repository first,
  by a transfer to its private twin. If it has no private twin, transfer
  it to the central work repository.
- Add no owner line. Check parent links under item 5.
- Give the answerer of each question. Ask the operator only about
  operator intent.
- Resume from current records. Retry the close of an open item with a
  current drop record. Skip other current records unless rechecking a split
  parent whose children cover its outcome.
- Post the question comment as C. Refinement specifies. On requeue, read the
  prior records and use any replies and body edits to record which questions
  have answers. Reuse the question comment for gaps that still need answers.
- A split parent needs its own current ready record to show Ready.
- Never refine an item whose body you wrote.
- For a tracker import, use a separate run from the one that mapped the
  staged body.
- Cite no private repository in a public item.
- On a 403 or 429 response, back off. Stop when the write budget runs
  out, and report where you stopped.

At the end, report the count of each outcome, the items you skipped and
why, and any check you couldn't run.
```

## Review checklist

The [batch reviewer](C-refinement.md#status-and-the-ready-signal) posts a pass or fail verdict on the batch Chore before it closes. In ordinary refinement, a pm or `sm` run closes the Chore on a pass. The sync job requires a passing verdict before any item in the batch can show Ready.

A refinement of one new item has no batch. Its Ready shows at once. Each week, a reviewer samples those records the same way, as if the week's records formed one batch.

For the tracker import, the reviewer samples staged files and their evidence before the kernel creates issues. See [E. Tracker import](E-import.md#review-and-creation).

- [ ] The reviewer reads every item in a batch of 10 or fewer. In a larger batch, it takes 10 items or a fifth at random, whichever is more, and adds each Drop, each Split, and each fix that edited the outcome or the checks.
- [ ] A sampled record covers each applicable readiness check with a result and evidence that meets the [evidence rules](C-refinement.md#evidence-rules).
- [ ] The reviewer runs at least one cited command per sampled item again, at the cited commit, and gets the same result.
- [ ] For item 4, the reviewer's own search doesn't turn up done work or a duplicate that the refiner missed. For a command Request, the reviewer checks only for an open duplicate Request.
- [ ] Context omits claims about status that age. It keeps the fixed facts described in the [run brief](#run-brief).
- [ ] On each sampled Ready item or staged file, every outcome and acceptance check states a result a reader can see. A check that says only that work exists or that someone did it fails review.
- [ ] On each sampled item, a Reproduction section gives steps a reader can follow. The exact placeholder `Find a reproduction and add it here.` requires Status `Needs info` or a staged `needs-info` outcome.
- [ ] Each sampled item follows rules 7 through 9 in [D. Issue writing](D-issue-writing.md#the-rules).
- [ ] A sampled Ready item has no semicolon in its prose, each of its list items ends with a period, and none of its sentences describes the issue itself.
- [ ] The reviewer runs Vale with the org configuration again on each sampled Ready item and its record. The rerun shows the warnings the record lists and, outside a short text, zero errors.
- [ ] Each sampled item sits in the repository the routing order picks. An internal-security item sits in a private repository.
- [ ] Each sampled item follows the [owner and role label rule in D. Issue writing](D-issue-writing.md#the-rules).
- [ ] The parent links on each sampled item pass readiness check 5.
- [ ] For each fix, the edit history shows that the item's scope didn't change.
- [ ] Each Needs info question gives an answerer, and none asks the operator about something a role could answer.
- [ ] Each `drop` cites the change, the duplicate, or the reason that has gone. For ordinary refinement, the close reason matches the record. For the import, the dropped item stays in staging.
- [ ] Each `split` child fits one pull request and goes to the right repository. For ordinary refinement, the child links to the original. For the import, the kernel creates only approved replacements.

A rubber-stamp Ready leaves signs, and one of them sends the reviewer to read the whole batch:

- More than three quarters of the drafts show a passing result for each readiness check without a fix.
- Two or more records share the same evidence text, or cite a commit without a command.
- A Ready item still matches a pattern of item 2, 3, 7, or 8 when checked again.
- An item 4 pass lacks an issue search.

The reviewer checks that every `ready` result records a pass or fix for each applicable check. In ordinary refinement, the reviewer also checks that the record is current. The reviewer checks the sample's evidence and inspects the whole batch if there are signs of a rubber stamp. One failed check on any reviewed item fails the batch. A failed batch gets a new refinement run.

## Failure modes

| Failure | Effect | Safeguard |
| --- | --- | --- |
| A refiner marks items Ready without checking | Workers start on unclear or done work | The sampled review and the rubber-stamp signs |
| A fix changes the item's scope | The work no longer matches the author's intent | The fix rules, and the reviewer's look at the edit history |
| The DoR turns into a phase gate | Work waits for detail a worker could find while working | The DoR doesn't require a design or an estimate, exploration splits off as a Spike, and fixes happen in place |
| A check says only that someone did the work | Someone closes the item without a result anyone saw | Item 3, the existence patterns of D, and the review checklist |
| Someone edits a Ready item or ticks a check | The record no longer describes the body | A change to the body hash makes the record stale. The sync job sets Status to `Needs refinement`. |
| Someone sets Status to `Ready` by hand | An item looks ready without a record | The sync job derives Status from the issue state and the refinement record |
| An assignee or open pull request appears before refinement | An item could appear to be in progress before it passes the gate | In progress requires the item's own current `ready` record, and any batch it belongs to must pass review |
| A built-in Project workflow writes Status | Status disagrees with the records | The sync job's setup turns off each built-in workflow that sets Status |
| Needs info piles up for the operator | Operator time goes to triage | The rule that the operator answers only intent, and D8 |
| Someone answers a Needs info question in a comment. | A comment alone leaves the item at Needs info. | The [question comment](C-refinement.md#outcome-writes) gives the steps to return the item to the queue. The next refinement record cites the answer. |
| A run stops partway through a batch | Part of a batch has records | Runs resume from current records and retry any pending drop close. |
| An internal-security item sits in a public repository | Anyone can read about the weakness | The security-topic warning of D, item 5, and a transfer before any other write |
| Vale's document-wide rules flag a short body | An honest short text can't pass | Warnings only below the short threshold |
| A record cites private evidence on a public item | A private detail leaks | The kernel's private-detail scan refuses the write |
| Merged work makes a Ready item moot later | Someone repeats done work | The worker checks item 4 again before opening a pull request |
