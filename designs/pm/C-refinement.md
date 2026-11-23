---
type: Design Appendix
title: Project management issue refinement
description: The Definition of Ready, the refinement procedure that runs before an issue or service request is ready to work, and how the tracker import passes through it.
tags: [pm, refinement, ready, import, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Project management issue refinement

Back to [the design](README.md).

Every issue and every service request passes refinement before anyone starts work on it, under the operator's ruling recorded as D6 in [decisions](decisions.md). A refinement run checks the item against the Definition of Ready (DoR) and fixes what it may. It then records one of five outcomes with evidence. The whole process is new, and none of it exists yet.

## Scope

The gate covers the six work item variants of [the content model](A-content-model.md), from Initiative down to Spike, and the Requests of the service-management records design, which the service-management role (`sm`) refines. Incidents differ, and D9 puts the question of whether they skip the gate to the operator.

Problem and change records follow the same seven items under `sm`. Their mapping arrives with their public appendices in the service-management records design.

## Status and the Ready signal

The README rule stays as written: only the sync job writes the Status field. Refinement never touches that field. Instead, the refine verb writes GitHub state, and the sync job derives every Status value from it. D7 in [decisions](decisions.md) records this choice.

| Status | The sync job sets it when |
| --- | --- |
| Needs refinement | The item is open and lacks a current refinement record |
| Needs info | The latest current record has the needs-info outcome |
| Ready | The latest current record has the ready outcome, its batch, if any, passed review, and no open pull request links the item |
| In progress | An open pull request links the item, or an assignee has it |
| Done | GitHub closed the item as completed |
| Dropped | GitHub closed the item as not planned or as a duplicate |

A record counts as current under three conditions. First, the kernel's account wrote it through the refine verb. Second, the item bears the refinement label that the record gives, and third, the body hash in the record matches the body now. A label added by hand, or a record that a later edit made stale, leaves the item at Needs refinement.

After a split, the parent takes its Status from its children. It shows Ready once each child shows Ready or a closed state, and In progress when any child is in progress.

Only a refinement run may make an item Ready, and only through the refine verb. Work items go to a run of the pm role, while Requests go to a run of the `sm` role. The run that wrote an item's body can't refine that item, and the reviewer of a batch can't refine items in it. That rule keeps the maker and the checker apart, as the agent roles registry requires.

Deriving Ready from GitHub state leaves one writer for the field and one audit trail in the issue itself. Anyone who edits the field by hand sees the next sync run put it back. Labels and records remain readable in GitHub while the Project is down.

## Readiness checks

Each item below says how its check runs and what runs it, using these kinds of check:

- A schema check is a Configure Unify Execute (CUE) rule over the block model.
- A kernel check needs live data, so it runs in the kernel's named list.
- A repo read means the refiner runs commands against a commit that the record cites.
- Judgment means the refiner decides and writes down why.

1. **The outcome is one sentence about the state after the work.** The variant's outcome section contains exactly one sentence, and that sentence describes a state the work leaves behind. For a Defect, the expected behavior counts as its outcome, and for a Spike, the decision it unblocks. Checked by a schema check for the sentence count, plus judgment that a stranger could observe the state.
2. **The context makes sense to a stranger.** It lacks retired notes, run names, process dates, line numbers, bare issue references, and text about where the content came from. It identifies the repository, the component, and the current behavior. Checked by a schema check of patterns in the issue base for the mechanical forms, plus judgment that someone new to the work could act on it.
3. **The checks are specific and testable, within the variant's caps.** Each check gives one observable result and how to see it, as a command, a path, or a visible behavior. Vague words such as works well, cleaner, or improved fail it. Checked by a schema check for the task list and the caps, plus judgment that a second person could run each check and agree on the result.
4. **Nobody has done the work, and no other open issue covers it.** A search at the target repository's current commit doesn't find a merged change that already meets the outcome. The refiner also searches open issues in the target repository, in its twin when the run can read it, and in the central work repository. Checked by a repo read and an issue search. The record cites the command and result count of each.
5. **Relations, fields, and labels are valid for the variant.** The item is in the repository that the [routing order](B-routing-and-taxonomy.md) picks. Its parent has the right type, and each blocked-by link points at an issue that exists. Priority, Source, Work type, and Severity for a Defect have allowed values, Handler appears only on a Request, and the sync job has given it a Status. Each label comes from the label configuration in the tools repository. Checked by a kernel check against live data, plus judgment for any value the refiner fills in.
6. **One pull request can deliver it under the size limits.** The refiner lists the paths the change likely touches and estimates its size against the target repository's `pr_size` limits, which in this repository are 400 code lines and 1,500 raw lines. Work that spans repositories fails it, and an estimate over half a limit fails it too. An Epic or Initiative passes when its children together cover its outcome. Checked by judgment, backed by a repo read of the listed paths.
7. **Each question has an answer.** The text lacks markers such as TBD or TODO, and it asks nothing outside a Spike's question section. A reader's likely questions find their answers in the text, or the item moves to Needs info with the answerer given. The operator answers only questions of operator intent, and a role takes the rest. Checked by a schema check for the markers, plus judgment for the unwritten questions.

The schema refuses a create or an edit at once, so new items arrive with items 1, 2, 3, and 7 partly checked. Imported items use a looser variant, described under the import gate, and refinement does that work for them.

## The procedure

### Runs and inputs

A refinement run is a headless agent run that a pm or `sm` run launches, for one item or for a batch of at most 25. It doesn't keep state of its own. Its results live in the items, as records, labels, links, and edits, and in the batch Chore when there is one. A run that stops partway resumes safely, because it skips each item that already has a current record.

Its inputs are the issue list or query, the commit of each target repository, and the label configuration. It also receives the variant guides from `tbhbdev pm guide`. Every write goes through the kernel, so the secret scan and the private-detail routing apply to each comment and edit.

### Steps for each item

1. Read the item and its comments. Then read the linked issues: parent, children, and blocked-by links.
2. Run the schema checks with `tbhbdev pm refine --check`, which reports each finding without a write.
3. Walk items 1 to 7 in order and write one result for each, with evidence. A result is pass, fixed, or fail.
4. Decide the outcome from the results, using the table that follows.
5. Write the outcome through `tbhbdev pm refine`, which posts the record and applies the labels, links, closes, and edits in one call.

| Outcome | When it applies |
| --- | --- |
| Ready | Each of the seven items passes as written |
| Fixed, then Ready | Some items failed, the refiner fixed each one without changing the item's scope, and each now passes |
| Needs info | A fix would change intent, or a question needs an answer the run can't find |
| Split | Item 6 fails, and the work divides into pieces that each pass alone |
| Drop | Item 4 shows the work done or covered by a duplicate, or the reason for it has gone |

A fix may do any of these:

- Restate, trim, or move text between sections.
- Remove stale context.
- Fill an empty field from evidence.
- Correct a link.
- Convert an imported item into its real variant.

A fix may not change the meaning of the outcome sentence or the intent of a check. It may not change a Priority that someone already set either. Each of those changes makes the outcome Needs info.

### Outcome writes

Each outcome writes one record comment. It goes through the refinement record variant. The record lists items 1 to 7 with a result and evidence for each, then the outcome and the refining run. It ends with the batch Chore, if any, and a Secure Hash Algorithm (SHA) hash, SHA-256, of the body after any fixes. A batch run writes one record per item, never one for the whole batch.

- **Ready** writes the record and the ready label of the refinement family. It edits nothing else.
- **Fixed, then Ready** writes the body and field edits first, then the record and the ready label. The record lists each fix and the item it served, and GitHub's edit history keeps the earlier text.
- **Needs info** writes the record and the needs-info label, plus one numbered question per gap. Each question gives its answerer, either a role through the role label family or the operator for a question of intent. D8 covers how the operator receives those questions.
- **Split** creates each child through the kernel in the right repository and links it as a sub-issue. When the children are Features, it retypes the original to an Epic. It then writes the record and the split label on the original, and each child gets its own refinement and record later or in the same run.
- **Drop** writes the record and then closes the item. Merged work that already met it closes it as completed. A duplicate closes it as a duplicate, with a link, and a reason that has gone closes it as not planned.

The tools label configuration spells each label as `family/value` and lists the exact values. This appendix needs only three outcomes from the refinement family: ready, needs-info, and split.

An answer on a Needs info item adds a comment, and the next refinement pass picks the item up from its Needs info Status. An edit to a Ready item's body changes its hash, and the sync job moves the item back to Needs refinement.

### Evidence rules

Each claim in a record cites at least one of these:

- A commit as a full or 12-character SHA, with its repository.
- A pull request as `owner/repo#number`.
- A path at a commit, as `path@sha`.
- A command, the commit or query it ran against, and its result count or a one-line result.

A pass on item 4 always cites both a repo search and an issue search. A record on a public item never cites a private repository. When the only evidence is private, the refiner writes it on the private twin. The public record then says that one private citation exists. A review fails an unsupported claim, even one with a correct conclusion.

## Service requests

The `sm` role applies the same seven items to Requests. In the service-management records design, an Agent Request goes to an agent role, while the two kinds of operator Request, approval and command, go to the operator.

| Item | Agent Request | Approval Request | Command Request |
| --- | --- | --- | --- |
| 1. Outcome | The ask says what's true once answered | The decision says what changes on yes | The Changes section says what the script changes |
| 2. Context | Applies | Applies | Applies, and the reason an agent can't run it makes sense alone |
| 3. Checks | Applies when the ask is a change | Each option states its effect | Checks first, Exit codes, Undo, and Dry run are each testable |
| 4. Not done | Applies | Applies, including a past Decision on the same question | Applies, including an earlier complete Request |
| 5. Fields | Handler Agent, one area label, and Priority | Handler Operator and Priority, without an area label | Handler Operator and a script hash, without an area label |
| 6. Size | Applies when the ask is a change | Doesn't apply | One script, checked against the command pattern |
| 7. Questions | Applies | Applies, and the Request itself is the question to the operator | Applies |

Severity and Work type never apply to a Request, and parent and blocked-by links apply only when they exist. Also, an operator Request whose body doesn't give a reason that an agent account can't do the work fails item 5 and goes back to its author.

For an operator Request, refinement runs while the Request is open, before the workflow moves it to waiting on operator. That workflow gains one condition for that move: a current ready record. The `sm` role makes that change in the service-management records design.

## Import gate

Plan step 7 imports about 700 issues from the earlier tracker. Each one enters through the kernel in an imported variant. That variant keeps the title and one Context section of the original text, and it runs the scans without section or cap rules.

The import sets Source to its import value. Each imported item then shows Needs refinement, and nobody can start it until refinement turns it into a real variant. The import uses issues and the org Project, without a new tracker. One Epic in the central work repository covers the import, and each batch is a Chore under it with its query and outcome counts.

### Order and batches

1. Order by the type that the import gives each item. Initiatives and Epics come first, then Features and Spikes, then Defects and Chores.
2. Within a level, take `Expedite` before `Standard` before `Intangible`, then the oldest first.
3. Wait to refine a child until its parent has an outcome. A child under a dropped parent gets a new parent or a Drop of its own.
4. Keep each batch to one target repository and at most 25 items.

### Throttling

GitHub limits content writes per minute and per hour. A refinement writes two to five times per item. For 700 items that comes to a few thousand writes. Refinement runs share one budget of at most 200 writes an hour, with one run per target repository at a time, which leaves room for normal work under GitHub's limits. This design hasn't measured those limits yet.

Review sets the pace as well. At most two batches wait for review at once, and the launcher doesn't start another batch until one of them clears review.

### Calibration

Before the first real batch, a run refines 20 imported items in a test repository. The set includes known bad cases, among them a done item, a duplicate, an item too large for one pull request, and stale context. The pm role compares the outcomes with the known answers and tunes the brief before the import starts.

### Finish line

The import ends when four counts hold:

- Zero imported items show Needs refinement.
- Each Needs info item gives its answerer.
- Each batch Chore has closed after a passing review.
- The import Epic's closing comment gives the outcome counts.

A Needs info item without an answer after 30 days gets one reminder. After that, the pm role may drop it as not planned, citing the unanswered question in the record. Items that wait on the operator are exempt.

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
Write budget: <writes per hour>, shared with other refinement runs

For each item, in the order given:
1. Read the item, its comments, its parent, its children, and its
   blocked-by links.
2. Run `tbhbdev pm refine --check <item>` and read every finding.
3. Walk items 1 to 7 of the DoR. For each, write pass, fixed, or fail,
   with evidence: a SHA, a pull request, a path at a SHA, or a command
   with the commit or query it ran against and its result.
4. Choose one outcome: ready, fixed-then-ready, needs-info, split, or drop.
5. Write it with `tbhbdev pm refine <item> --outcome <outcome>`.

Rules:
- Fix wording, stale context, empty fields, and links. Never change the
  item's scope; ask instead, as needs-info.
- Item 4 needs a repo search and an issue search, both cited.
- Give the answerer of each question. Ask the operator only about
  operator intent.
- Skip any item that already has a current record.
- Never refine an item whose body you wrote.
- Cite no private repository in a public item.
- Stop when the write budget runs out, and report where you stopped.

At the end, report the count of each outcome, the items you skipped and
why, and any check you couldn't run.
```

## Review checklist

A second run reviews each batch before its Chore closes. It reads as a reviewer only, from a model family other than the refiner's, and posts its verdict as a comment on the batch Chore. On a pass, a pm or `sm` run closes the Chore. On a fail, that run removes the ready labels in the batch.

A refinement of one new item has no batch. Its Ready shows at once. Each week, a reviewer samples those records the same way, as if the week's records formed one batch.

- [ ] The reviewer reads every item in a batch of 10 or fewer. In a larger batch, it takes 10 items or a fifth at random, whichever is more, and adds each Drop, each Split, and each fix that edited the outcome or the checks.
- [ ] A sampled record covers items 1 to 7, each with a result and evidence that meets the evidence rules.
- [ ] The reviewer runs at least one cited command per sampled item again, at the cited commit, and gets the same result.
- [ ] For item 4, the reviewer's own search doesn't turn up done work or a duplicate that the refiner missed.
- [ ] For each fix, the edit history shows that the item's scope didn't change.
- [ ] Each Needs info question gives an answerer, and none asks the operator about something a role could answer.
- [ ] Each Drop cites the change, the duplicate, or the reason that has gone, and the close reason matches it.
- [ ] Each Split child is in the right repository and links to the original. It fits one pull request.

A rubber-stamp Ready leaves signs, and one of them sends the reviewer to read the whole batch:

- Over three in four imported items come out Ready without a fix, though imported text rarely passes item 2 unedited.
- Two or more records share the same evidence text, or cite a commit without a command.
- A Ready item still matches a pattern of item 2 or item 7 when checked again.
- An item 4 pass lacks an issue search.

The batch fails when two or more sampled items fail, or when a full read confirms a rubber stamp. A failed batch gets a new refinement run, never the same one.

## Failure modes

| Failure | Effect | Safeguard |
| --- | --- | --- |
| A refiner marks items Ready without checking | Workers start on unclear or done work | The sampled review and the rubber-stamp signs |
| A fix changes the item's scope | The work no longer matches the author's intent | The fix rules, and the reviewer's look at the edit history |
| Someone edits a Ready item | The record no longer describes the text | The body hash, which returns the item to Needs refinement |
| Someone adds the ready label by hand | An item looks ready without a record | The sync job ignores a label without a current record |
| Needs info piles up for the operator | Operator time goes to triage | The rule that the operator answers only intent, and D8 |
| A run stops partway through a batch | Part of a batch has records | Runs skip items with a current record and resume |
| A record cites private evidence on a public item | A private detail leaks | The kernel's private-detail scan refuses the write |
| Merged work makes a Ready item moot later | Someone repeats done work | The worker checks item 4 again before opening a pull request |

## Limits of refinement

- Refinement doesn't set or rank Priority. It fills only an empty Priority with `Standard`, and any other change needs the priority verb with a work note.
- It doesn't estimate effort or assign anyone, and it doesn't start work.
- Approving or denying a Request stays with the operator's request commands.
- Operator intent comes from the operator. Refinement asks.
- The Status field belongs to the sync job alone.
- Issues and the org Project remain the only tracker.

## Changes to the rest of the design

- The README's Status rule stays. This appendix adds the six Status values and the GitHub state behind each one.
- The taxonomy gains a Source field. Work type already appears in the taxonomy schema.
- The verbs gain `refine` with an edit operation. The variants gain the imported variant and the refinement record variant.
- The README and B spell labels as `area:<repo>`. The tools label configuration writes `family/value`, and both files take its spelling once it merges.
- The service-management records design gains one condition before waiting on operator, as the service requests section describes.
