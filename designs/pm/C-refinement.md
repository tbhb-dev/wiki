---
type: Design Appendix
title: Project management issue refinement
description: The Definition of Ready, the refinement procedure that runs before an issue or service request is ready to work, and the sources behind those choices.
tags: [pm, refinement, ready, import, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Project management issue refinement

Back to [the design](README.md).

Every issue and every service request passes refinement before anyone starts work on it, under the operator's ruling recorded as D6 in [decisions](decisions.md). A refinement run checks the item against the Definition of Ready (DoR) and fixes what it may.

It then records one of five outcomes with evidence. The whole process is new, and none of it exists yet.

## Why a gate

The Scrum Guide has no Definition of Ready. It treats refinement as ongoing work that adds detail, order, and size until a team can finish an item within one Sprint. This design makes readiness a gate anyway, because D6 says so and because an agent starts work without the conversation a team would have first.

A ready checklist can drift into a phase gate that holds work until every detail exists. This DoR requires only what a worker needs to start the work and to check the result.

It never requires a design, an effort estimate, or an assignee. Work that needs exploration first splits off a Spike.

## Scope

The gate covers the six work item variants of [the content model](A-content-model.md), from Initiative down to Spike. Requests from the service-management records design pass it too, under the service-management role (`sm`).

Incidents work differently. Whether they skip the gate is a question for the operator in D9.

Problem and change records follow the same eight items under `sm`, and their mapping arrives with their public appendices in the service-management records design. The `sm` role is still designing the Problem type, so this appendix only leaves room for it:

- A Defect may have a Problem or an Epic as its parent, or no parent at all.
- A Problem is likely to become the parent of the Defects and Chores that fix it, once the `sm` role settles the Problem type and its fields.
- An Incident links to its Problem.

## Status and the Ready signal

The README rule stays as written: only the sync job writes the Status field. Refinement never touches that field. Instead, the refine verb writes GitHub state, and the sync job derives every Status value from it, as D7 in [decisions](decisions.md) records.

| Status | The sync job sets it when |
| --- | --- |
| Needs refinement | The item is open and lacks a current refinement record |
| Needs info | The latest current record has the needs-info outcome |
| Ready | The latest current record has the ready outcome, its batch, if any, passed review, and no open pull request links the item |
| In progress | An open pull request links the item, or an assignee has it |
| Done | GitHub closed the item as completed |
| Dropped | GitHub closed the item as not planned or as a duplicate |

A record counts as current under three conditions. First, the kernel's account wrote it through the refine verb. Second, the item bears the refinement label that the record gives, and third, the body hash in the record matches the body now.

A label added by hand, or a record that a later edit made stale, leaves the item at Needs refinement. After a split, the parent takes its Status from its children. It shows Ready once each child shows Ready or a closed state, and In progress when any child is in progress.

Built-in workflows of the org Project can also write Status. Two of them start enabled, and each sets Done, one when an item closes and one when a pull request merges. The sync job's setup turns off every built-in workflow that sets Status, which leaves the sync job as the only writer.

Issue and pull request events start the sync job, and so does a schedule. GitHub doesn't start a new workflow run for an event that a workflow's own token caused, and the schedule catches those changes.

Only a refinement run may make an item Ready, and only through the refine verb. Work items go to a run of the pm role, while Requests go to a run of the `sm` role. The run that wrote an item's body can't refine that item, and the reviewer of a batch can't refine items in it, which keeps the maker and the checker apart as the agent roles registry requires.

A worker who finds a Ready item unclear doesn't start it. Then the worker removes the ready label with `tbhbdev pm refine --return` and one comment that states the gap. After that, the record stops counting as current, and the sync job shows Needs refinement.

Deriving Ready from GitHub state leaves one writer for the field and one audit trail in the issue itself. Anyone who edits the field by hand sees the next sync run put it back. Labels and records remain readable in GitHub while the Project is down.

## Readiness checks

Each item below says how its check runs and what runs it, using these kinds of check:

- A schema check is a Configure Unify Execute (CUE) rule over the block model.
- A kernel check needs live data, so it runs in the kernel's named list.
- A repo read means the refiner runs commands against a commit that the record cites.
- Judgment means the refiner decides and writes down why.

1. **The outcome is one sentence about an observable state after the work.** The variant's outcome section contains exactly one sentence. That sentence describes a state the work leaves behind, which someone can observe. It never says only that the work exists or that someone did it. For a Defect, the expected behavior counts as its outcome, and for a Spike, the decision it unblocks. Checked by a schema check for the sentence count and the existence patterns of [D. Issue writing](D-issue-writing.md), plus judgment that a stranger could observe the state.
2. **The context makes sense to a stranger.** It lacks retired notes, run names, process dates, line numbers, bare issue references, and text about where the content came from. It identifies the repository, the component, and the current behavior. Checked by a schema check of patterns in the issue base for the mechanical forms, plus judgment that someone new to the work could act on it.
3. **Each check states an observable result, within the variant's caps.** Each check states one result that a person or another system can see, and how to see it, such as a command and its output or a path and its content. A check fails when it says only that the work exists or that someone did it, as in a spike that exists, a pull request that merged, or a doc that changed. Vague words such as works well, cleaner, or improved fail it too. Checked by a schema check for the task list, the caps, and the existence and vague-result patterns of D, plus judgment that a second person could run each check and agree on the result.
4. **Nobody has done the work, and no other open issue covers it.** A search at the target repository's current commit doesn't find a merged change that already meets the outcome. The refiner also searches open issues in the target repository, in its twin when the run can read it, and in the central work repository. Checked by a repo read and an issue search. The record cites the command and result count of each.
5. **Relations, fields, and labels are valid for the variant.** The item is in the repository that the [routing order](B-routing-and-taxonomy.md) picks. Its parent has a type the variant allows. A Defect's parent is a Problem or an Epic, and a Defect may also have no parent. Each blocked-by link points at an issue that exists. Priority, Source, Work type, and Severity for a Defect have allowed values, Handler appears only on a Request, and the sync job has given it a Status. No item has an owner field, because the repository implies the responsible role. A `role/<role>` label appears only when the work belongs to another role. Each label comes from the label configuration in the tools repository. An item about internal security is in a private repository, as the [internal security](#internal-security-topics) section describes. Checked by a kernel check against live data, which includes the parent-type check of A. The owner-line and security-topic checks of D also run, plus judgment for any value the refiner fills in.
6. **One pull request can deliver it under the size limits.** The refiner lists the paths the change likely touches and estimates its size against the target repository's `pr_size` limits, which in this repository are 400 code lines and 1,500 raw lines. Work that spans repositories fails it, and an estimate over half a limit fails it too. An Epic or Initiative passes when its children together cover its outcome. Checked by judgment, backed by a repo read of the listed paths.
7. **Each question has an answer.** The text lacks markers such as TBD or TODO, and it asks nothing outside a Spike's question section. A reader's likely questions find their answers in the text, or the item moves to Needs info with the answerer given. The operator answers only questions of operator intent, and a role takes the rest. A question that needs exploration to answer becomes a Spike through a split. Checked by a schema check for the markers, plus judgment for the unwritten questions.
8. **Every sentence states one fact about the work.** Each sentence states one fact in plain subject-verb order, and no semicolon or comma splice joins two facts. Each list item is a full sentence that ends with a period. No sentence describes the issue itself or its tracking, links, or history. GitHub's sub-issue and dependency links record those. The [Vale pass](#vale-pass) with the org configuration doesn't report an error on the body. Checked by the semicolon, final-period, and meta-exposition patterns of D and by the Vale pass. Judgment covers comma splices and whether a stranger reads each sentence once.

The schema refuses a create or an edit at once, so new items arrive with items 1, 2, 3, 7, and 8 partly checked. Imported items use a looser variant, described in [E. Tracker import](E-import.md), and refinement does that work for them.

A person can still open an issue in the GitHub web interface, which skips the kernel. An issue form can set the type, labels, and Project of such an issue.

It can't run the schema. Refinement is the gate for those issues: `refine --check` runs the same schema checks, and the issue shows Needs refinement until a run converts it into its real variant.

## The procedure

### Runs and inputs

A refinement run is a headless agent run that a pm or `sm` run launches, for one item or for a batch of at most 25. It doesn't keep state of its own.

Its results live in the items, as records, labels, links, and edits, and in the batch Chore when there is one. A run that stops partway resumes safely, because it skips each item that already has a current record.

Its inputs are the issue list or query, the commit of each target repository, and the label configuration. It also receives the variant guides from `tbhbdev pm guide`. Every write goes through the kernel, so the secret scan and the private-detail routing apply to each comment and edit.

[F. Refinement runs](F-refinement-runs.md) gives the run brief, the review checklist, and the failure modes.

### Steps for each item

1. Read the item and its comments. Then read the linked issues: parent, children, and blocked-by links.
2. Run the schema checks with `tbhbdev pm refine --check`, which reports each finding and each warning without a write.
3. Walk items 1 to 8 in order and write one result for each, with evidence. A result is pass, fixed, or fail.
4. Decide the outcome from the results, using the table that follows.
5. Run the [Vale pass](#vale-pass) on the final body and on each comment the outcome posts, and fix each error before any write.
6. Write the outcome through `tbhbdev pm refine`, which posts the record and applies the labels, links, closes, and edits in one call.

| Outcome | When it applies |
| --- | --- |
| Ready | Each of the eight items passes as written |
| Fixed, then Ready | Some items failed, the refiner fixed each one without changing the item's scope, and each now passes |
| Needs info | A fix would change intent, or a question needs an answer the run can't find |
| Split | Item 6 fails and the work divides into pieces that each pass alone, or item 7 needs a Spike first |
| Drop | Item 4 shows the work done or covered by a duplicate, or the reason for it has gone |

A fix may do any of these:

- Restate, trim, or move text between sections.
- Break a sentence into one fact per sentence, or into bullets.
- End each list item with a period.
- Delete a sentence that describes the issue itself.
- Rewrite a check that says only that someone did the work into the observable result the item's text already implies.
- Remove stale context.
- Fill an empty field from evidence.
- Correct a link.
- Transfer an internal-security item from a public repository to a private one, before any other write.
- Convert an imported item into its real variant.

A fix may not change the meaning of the outcome sentence or the intent of a check. It may not change a Priority that someone already set either. Each of those changes makes the outcome Needs info, and so does a check whose observable result the text doesn't make plain.

### Outcome writes

Each outcome writes one record comment through the refinement record variant. The record lists items 1 to 8 with a result and evidence for each, then the outcome and the refining run. It ends with the batch Chore, if any, and a Secure Hash Algorithm (SHA) hash, SHA-256, of the body after any fixes.

A batch run writes one record per item. It never writes one record for a whole batch.

- **Ready** writes the record and the ready label of the refinement family. It edits nothing else.
- **Fixed, then Ready** writes the body and field edits first, then the record and the ready label. The record lists each fix and the item it served, and GitHub's edit history keeps the earlier text.
- **Needs info** writes the record and the needs-info label, plus one numbered question per gap. Each question gives its answerer, either a role through the role label family or the operator for a question of intent. D8 covers how the operator receives those questions.
- **Split** creates each child through the kernel in the right repository and links it as a sub-issue. When the children are Features, it retypes the original to an Epic. It then writes the record and the split label on the original, and each child gets its own refinement and record later or in the same run.
- **Drop** writes the record and then closes the item. Merged work that already met it closes it as completed. A duplicate closes it as a duplicate, with a link, and a reason that has gone closes it as not planned.

The tools label configuration spells each label as `family/value` and lists the exact values. This appendix needs only three outcomes from the refinement family: ready, needs-info, and split.

An answer on a Needs info item adds a comment, and the next refinement pass picks the item up from its Needs info Status. An edit to a Ready item's body changes its hash, and the sync job moves the item back to Needs refinement.

### Vale pass

Each refinement run checks prose with Vale, using the organization's [`.vale.ini`](https://github.com/tbhb-dev/rules/blob/main/.vale.ini) and [`.vale/`](https://github.com/tbhb-dev/rules/tree/main/.vale) styles at the rules repository's current commit. This design copies none of their rules.

The run lints the final body before writing it, and each comment it posts before posting, the record included. Vale reads each as Markdown, as in `vale --ext=.md`.

An error from the configuration blocks the write of a body or comment at the short threshold or longer. In a short text, the run reports each error as a warning instead. A warning never blocks, and the refiner fixes it or keeps it with a one-line reason in the record.

A body counts as short when its prose runs under half the total-word cap of its variant in [A](A-content-model.md), such as 100 words for a Feature or 65 for a Spike. Comments have no cap of their own, so their threshold is 65 words, half of the Spike's 130, the smallest cap in A.

Why the threshold? Some error-level rules measure spread across a whole document, and a few sentences give them too little to measure.

Vale reports findings and leaves the text alone.

An error that blocks the write sends the refiner back to fix the text under the fix rules and run Vale again. If the only fix would change intent, the outcome becomes Needs info and the body keeps its earlier text. Every change to the text is a fix that the refiner makes by hand and the record lists, because the run doesn't apply a suggested replacement on its own.

Findings come back as a list of line, rule, level, and message, as in `12: Google.Semicolons (error) Use semicolons judiciously.` The record lists them under item 8.

If the Vale binary or the configuration is missing, the run fails closed. D10 in [decisions](decisions.md) asks the tools role to move this pass into the content check.

### Evidence rules

Each claim in a record cites at least one source, in one of these forms:

- A commit appears as a full or 12-character SHA, with its repository.
- A pull request appears as `owner/repo#number`.
- A path appears at a commit, as `path@sha`.
- A command appears with the commit or query it ran against, and with its result count or a one-line result.

A pass on item 4 always cites both a repo search and an issue search. A record on a public item never cites a private repository. When the only evidence is private, the refiner writes it on the private twin.

The public record then says that one private citation exists. A review fails an unsupported claim, even one with a correct conclusion.

## Internal security topics

Every issue about internal security goes in a private repository. Internal security means a weakness in the organization's own systems, such as a vulnerability, a gap in access control, an exposed credential path, or a missing detection. A public advisory about an upstream dependency doesn't count.

If refinement finds such an item in a public repository, its first write transfers the item to the private twin. Where no twin exists, the item goes to the central work repository.

GitHub's transfer docs allow a move between repositories of one organization. They rule out only a move from a private repository to a public one.

The run writes the record and any fixes only after the move, so no record repeats the detail in public. A transfer hides the item from public view, but it can't recall copies made while the item was public. The record on the moved item says how long the item was public.

## Service requests

The `sm` role applies the same eight items to Requests. In the service-management records design, an Agent Request goes to an agent role, while the two kinds of operator Request, approval and command, go to the operator.

| Item | Agent Request | Approval Request | Command Request |
| --- | --- | --- | --- |
| 1. Outcome | The ask says what's observably true once answered | The decision says what changes on yes | The Changes section says what the script changes |
| 2. Context | Applies | Applies | Applies, and the reason an agent can't run it makes sense alone |
| 3. Checks | Applies when the ask is a change | Each option states its effect | Checks first, Exit codes, Undo, and Dry run are each testable |
| 4. Not done | Applies | Applies, including a past Decision on the same question | Applies, including an earlier complete Request |
| 5. Fields | Handler Agent, one area label, and Priority | Handler Operator and Priority, without an area label | Handler Operator and a script hash, without an area label |
| 6. Size | Applies when the ask is a change | Doesn't apply | One script, checked against the command pattern |
| 7. Questions | Applies | Applies, and the Request itself is the question to the operator | Applies |
| 8. Sentences | Applies | Applies | Applies to the prose, never to the script |

Severity and Work type never apply to a Request, and parent and blocked-by links apply only when they exist. Also, an operator Request whose body doesn't give a reason that an agent account can't do the work fails item 5 and goes back to its author.

For an operator Request, refinement runs while the Request is open, before the workflow moves it to waiting on operator. That workflow gains one condition for that move: a current ready record. The `sm` role makes that change in the service-management records design.

## Limits of refinement

- Refinement doesn't set or rank Priority. It fills only an empty Priority with `Standard`, and any other change needs the priority verb with a work note.
- It doesn't estimate effort, assign anyone, add an owner, or start work.
- Approving or denying a Request stays with the operator's request commands.
- Operator intent comes from the operator. Refinement asks.
- The Status field belongs to the sync job alone.
- Issues and the org Project remain the only tracker.

## Changes to the rest of the design

- The README's Status rule stays. This appendix adds the six Status values and the GitHub state behind each one.
- The org Project's setup turns off each built-in workflow that sets Status, including the two that start enabled.
- The taxonomy gains a Source field. Work type already appears in the taxonomy schema.
- Issues have no owner field. The repository implies the responsible role, and a `role/<role>` label marks each exception.
- The verbs gain `refine` with an edit operation and a return operation. The variants gain the imported variant and the refinement record variant.
- A's parent-type kernel check accepts a Problem, an Epic, or no parent for a Defect.
- The routing order in B gains a rule that sends each internal-security item to a private repository.
- The content check in tools runs the Vale pass if the tools role agrees to D10. Until then, the refinement run and the import run call Vale themselves.
- The block model exposes the text of each paragraph and list item without its code spans, which the checks of D read.
- The README and B spell labels as `area:<repo>`. The tools label configuration writes `family/value`, and both files take its spelling once it merges.
- The service-management records design gains one condition before waiting on operator, as the service requests section describes.

## Sources

The web research pass of 2026-11-23 read these sources. Each line says what changed because of it, or what it confirmed.

- [The Scrum Guide (2020)](https://scrumguides.org/scrum-guide.html) has no DoR and treats refinement as ongoing. Added the Why a gate section and confirmed the size check of item 6.
- [Mountain Goat Software on the dangers of a DoR](https://www.mountaingoatsoftware.com/blog/the-dangers-of-a-definition-of-ready) warns that rules demanding finished upstream work form a stage gate. Added the limit that the DoR doesn't require a design, an estimate, or an assignee, and added the phase-gate failure mode in F.
- [Scrum.org on why the Scrum Guide omits the DoR](https://www.scrum.org/resources/blog/why-isnt-definition-ready-described-scrum-guide) makes the same point about overhead. Confirmed the phase-gate risk without a change.
- [Bill Wake's INVEST](https://xp123.com/articles/invest-in-good-stories-and-smart-tasks/) ties understanding a story to being able to test it. Confirmed items 3, 5, and 6 without a change.
- [The Gherkin reference](https://cucumber.io/docs/gherkin/reference/) says an outcome step checks what a user or outside system sees. Shaped items 1 and 3 so a check states what a person or another system can see.
- [GitHub's Copilot coding agent guidance](https://docs.github.com/en/copilot/tutorials/coding-agent/get-the-best-results) asks for a clear problem, acceptance criteria, and file pointers, and keeps exploratory work for people. Added the Spike split to item 7 and confirmed items 3 and 6.
- [Kubernetes issue triage](https://www.kubernetes.dev/docs/guide/issue-triage/) uses bot labels that change only by a command from a member, the author, or an assignee. Confirmed the label-driven states and the rule that ignores a hand-added label, without a change.
- [GitLab's handbook on how product work flows](https://handbook.gitlab.com/handbook/product-development/how-we-work/product-development-flow/) returns work to refinement when someone finds it incomplete after ready. Added the worker's return operation.
- [GitHub's issue form syntax](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-issue-forms) sets type, labels, and Project but checks only its own inputs. Added the paragraph on issues written outside the kernel.
- [GitHub's built-in Project workflows](https://docs.github.com/en/issues/planning-and-tracking-with-projects/automating-your-project/using-the-built-in-automations) include two that set Done and start enabled. Added the setup step that turns them off. Also added a failure mode in F and a change line.
- [GitHub's workflow token docs](https://docs.github.com/en/actions/concepts/security/github_token) say events the token causes don't start a new run. Added the scheduled run of the sync job.
- [GitHub's REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api) cap content creation at 80 a minute and 500 an hour. Replaced the unmeasured limit in E with these figures and added the per-minute budget and the backoff rule.
- [GitHub's sub-issue docs](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues) allow 100 sub-issues per parent. Added the line in E that the batch Chores fit under the import Epic.
- [GitHub's issue transfer docs](https://docs.github.com/en/issues/tracking-your-work-with-issues/administering-issues/transferring-an-issue-to-another-repository) allow a transfer within one owner and forbid a move from a private repository to a public one. Added the transfer of internal-security items.
- [GitHub's issue dependency docs](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/creating-issue-dependencies) show blocked-by links in Projects. Confirmed that links record relations, which backs rule 4 of D, without a change.
- [Google's style guide on lists](https://developers.google.com/style/lists) lets a fragment end without a period. Kept the stricter rule 3 of D, which doesn't allow fragments.
- [Digital.gov on short sentences](https://digital.gov/guides/plain-language/writing/clear-short) asks for one idea per sentence. Confirmed rule 2 of D without a change.
