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

# Refining issues for pm

Back to [the design](README.md).

Nobody starts work on an issue or service request until the item passes refinement under [decision D6](decisions.md#d6-must-issues-pass-refinement-before-work-starts). A refinement run checks the item against the Definition of Ready (DoR) and fixes what it may.

It then records one of four outcomes with evidence. The whole process is new, and none of it exists yet.

## Why a gate

The Scrum Guide has no Definition of Ready. It treats refinement as ongoing work that adds detail, order, and size until a team can finish an item within one Sprint. This design makes readiness a gate anyway, because D6 says so and because an agent starts work without the conversation a team would have first.

A ready checklist can drift into a phase gate that holds work until every detail exists. This DoR requires only what a worker needs to start the work and to check the result.

It never requires a design, an effort estimate, or an assignee. Work that needs exploration first splits off a Spike.

## Scope

The gate covers the six work item variants of [the content model](A-content-model.md), from Initiative down to Spike. The service-management role (`sm`) refines Requests, Incidents, Problems, and Changes. For each Incident, Problem, or Change, the refiner applies the common readiness checks below and uses the issue type's template and schema for its required sections, fields, and relations. Incidents follow this rule while D9 is open, and each Incident links to its Problem when one exists.

## Status and the Ready signal

Only the sync job writes Status on the issue. To supply the inputs, the refine verb writes records and issue state. See [decision D7](decisions.md#d7-how-does-an-item-become-ready-while-only-the-sync-job-writes-status).

A batch groups up to 25 items in one target repository under a batch Chore. An independent reviewer from a model family other than the refiner's posts the verdict on that Chore. For each `ready` record, the reviewer checks that the item meets the gate. All items that the reviewer examines under the [review checklist](F-refinement-runs.md#review-checklist) must also pass.

| Status | The sync job sets it when |
| --- | --- |
| Unverified | The kernel has yet to check an open issue created outside its write path. This status also applies to an imported issue that has never had a verified staging or refinement record. |
| Needs refinement | An open item that has cleared verification needs a current refinement record. This status also applies to open items with a current `split` record or a current `ready` record and a batch review that has yet to pass. |
| Needs info | The item is open and has a current `needs-info` record. An answer in a comment alone leaves the item at Needs info. A body edit or an explicit return to the queue makes the record stale. |
| Ready | The open item has its own current `ready` record. Any batch it belongs to has passed review. The item has neither a linked open pull request nor an assignee. |
| In progress | An assignee or a linked open pull request puts an open item with its own current `ready` record In progress. Any batch the item belongs to must have passed review. |
| Done | GitHub closed the item as completed. |
| Dropped | GitHub closed the item as not planned or as a duplicate. |
| No Status | The item is open and has a current `drop` record. The sync job clears Status until the close succeeds. |

No Status means the org issue field is empty. Only the latest refinement record from the kernel's account can be current. Ordinary refinement writes that record through the refine verb. A current record requires the item's body hash to match the record and no later return to the queue. A `ready` record also requires the live fields, labels, and relations to pass check 5.

For a tracker import, the kernel writes the staging result for each created issue on the batch Chore. The sync job can use that result as the issue's refinement record until ordinary refinement writes a newer record or returns the issue to the queue. The batch review must have passed, and the issue link and body hash must match. A `ready` result still requires check 5 to pass on live data. See [E. Tracker import](E-import.md#review-and-creation).

When check 5 fails, a current `needs-info` record still applies. Unrelated comments leave the record current. An assigned, refined item stays In progress after a routine progress comment.

When live fields, labels, or relations change, the sync job reruns check 5 for a `ready` record. Relations include the parent, sub-issues, and dependencies.

Ticking a check triggers a new readiness evaluation. The sync job rereads the body, including the check text and each checkbox state. It rereads the live labels and relations. It also rereads the org issue fields that check 5 uses for the variant. A changed body hash makes the record stale.

Removing a required parent from a Ready or In progress item makes its `ready` record stale. An item with a current `needs-info` record stays at Needs info after losing a required parent.

Ready and In progress still require a current record when the item has an assignee or a linked open pull request. After an ordinary refinement split, the refiner checks the parent again once the children together cover its outcome. The parent needs its own current `ready` record to show Ready or In progress.

During setup, the administrator turns off built-in Status workflows on both boards.

Issue, issue comment, and pull request events start the sync job, and so does a schedule. GitHub doesn't start a new workflow run for an event that a workflow's own token caused, and the schedule catches those changes.

Ordinary refinement supplies the Ready signal through the refine verb. Work items go to a run of the pm role, and service-management issues go to a run of the `sm` role. The agent roles registry keeps the maker and checker apart. The run that wrote an item's body can't refine it. A reviewer can't refine an item in the batch they review.

A worker who finds a Ready item unclear returns it with `tbhbdev pm refine --return` and gives the gap in a comment. The return makes the record stale and puts the item back at Needs refinement.

Deriving Ready from GitHub state leaves one writer for the field and one audit trail in the issue itself. Anyone who edits the field by hand sees the next sync run put it back. Fields and records remain readable on the issue while a Project is down.

## Readiness checks

Each item below says how its check runs and what runs it, using these kinds of check:

- A schema check is a Configure Unify Execute (CUE) rule over the block model.
- A kernel check needs live data, so it runs in the kernel's named list.
- A repo read means the refiner runs commands against a commit that the record cites.
- Judgment means the refiner decides and writes down why.

1. **The outcome is one sentence about an observable state after the work.** The variant's outcome section contains exactly one sentence. That sentence describes a state the work leaves behind, which someone can observe. It never says only that the work exists or that someone did it. For a Defect, the expected behavior counts as its outcome, and for a Spike, the decision it unblocks. Checked by a schema check for the sentence count and the existence patterns of [D. Issue writing](D-issue-writing.md), plus judgment that a stranger could observe the state.
2. **The context makes sense to a stranger.** It lacks retired notes, run names, process dates, line numbers, bare issue references, and mapping notes. It identifies the repository, the component, and the current behavior. Checked by a schema check of patterns in the issue base for the mechanical forms, plus judgment that someone new to the work could act on it.
3. **Each check states an observable result, within the variant's caps.** Each check states one result that a person or another system can see, and how to see it, such as a command and its output or a path and its content. A check fails when it says only that the work exists or that someone did it, as in a spike that exists, a pull request that merged, or a doc that changed. Vague words such as works well, cleaner, or improved fail it too. Checked by a schema check for the task list, the caps, and the existence and vague-result patterns of D, plus judgment that a second person could run each check and agree on the result.
4. **Nobody has done the work, and no other open issue covers it.** A search at the target repository's current commit doesn't find a merged change that already meets the outcome. The refiner also searches open issues in the target repository, in its twin when the run can read it, and in the central work repository. For a command Request, the refiner searches only for an open duplicate Request. The record cites each search command and its result count.
5. **Relations, org issue fields, and labels are valid for the variant.** The item is in the repository that the [routing order](B-routing-and-taxonomy.md#routing-order-for-create) picks. It has a parent if the variant requires one. A parent must have a type the variant allows. Each blocked-by link points at an issue that exists. Before closing a decision Request, the workflow checks for an `adr` link. Check fields and labels against the [shared taxonomy](B-routing-and-taxonomy.md#one-home-per-fact). An item about internal security is in a private repository, as the [internal security](#internal-security-topics) section describes. The kernel checks live data, including the [parent-type check](A-content-model.md#checks-outside-the-schema). The [owner-line and security-topic checks](D-issue-writing.md#content-checks) also run. The refiner judges any value they fill in.
6. **One pull request can deliver it under the size limits.** The refiner lists the paths the change likely touches and estimates its size against the target repository's `pr_size` limits, which in this repository are 400 code lines and 1,500 raw lines. Work that spans repositories fails it, and an estimate over half a limit fails it too. An Epic or Initiative passes when its children together cover its outcome. Checked by judgment, backed by a repo read of the listed paths. For a command Request, check that the text gives the command and its target host or repository. Check who approves it.
7. **The text answers what the reader needs to know.** The text lacks markers such as TBD or TODO. Use a Spike's Question section for the question the Spike investigates. In a decision Request, put the question to the operator in the decision text. If the reader needs an answer before work begins, move the item to Needs info and specify who can answer. Only the operator answers questions about their intent. Assign other questions to a role. Split off a Spike when the work needs exploration before it can start. The schema checks for markers. The refiner judges what the reader needs to know.
8. **Every sentence states one fact about the work.** Each sentence states one fact in plain subject-verb order, and no semicolon or comma splice joins two facts. Each list item is a full sentence that ends with a period. No sentence describes the issue itself or its tracking, links, or history. GitHub's sub-issue and dependency links record those. The [Vale pass](#vale-pass) with the org configuration doesn't report an error on the body. Checked by the semicolon, final-period, and meta-exposition patterns of D and by the Vale pass. Judgment covers comma splices and whether a stranger reads each sentence once.
9. **The Request identifies the approver.** The refiner checks the metadata for the Request variant under item 5.
10. **The Request specifies how to check the result.** The refiner checks that the Request says where to find the result and what shows success. A command Request uses the exit status and the output tail that the run posts on the Request.
11. **The title and outcome follow rule 7 of [D. Issue writing](D-issue-writing.md#the-rules).** The schema checks how the title starts. The refiner judges whether both describe the end state.
12. **Context follows rule 8 of [D. Issue writing](D-issue-writing.md#the-rules).** The refiner checks it by judgment.
13. **Decision work follows rule 9 of [D. Issue writing](D-issue-writing.md#the-rules).** The refiner checks the issue type and Handler.

The schema refuses a create or an edit at once, so new items arrive with items 1, 2, 3, 7, and 8 partly checked. For tracker imports, a separate run of the imports role refines the mapped files. The refinement run validates the destination type before creation. See [E. Tracker import](E-import.md#staging-and-refinement).

A person can still open an issue in the GitHub web interface, which skips the kernel. An issue form can set the type, labels, and Project of such an issue.

For those issues, `tbhbdev pm refine --check` runs the same schema checks. The sync job shows Unverified until the kernel checks the item.

## The procedure

### Runs and inputs

A pm or `sm` run launches a headless agent to refine one item or a batch. GitHub stores its state.

Refinement stores records, links, and edits on the items. For a batch, the run also records results in the Chore. On resumption, retry each pending close for an open item with a current `drop` record. Skip other current records. A follow-up run can revisit a split parent once the children together cover its outcome.

Its inputs are the issue list or query, the commit of each target repository, and the merged global and repository configuration for labels and issue fields. It also receives the variant guides from `tbhbdev pm guide`. Every write goes through the kernel, so the secret scan and the private-detail routing apply to each comment and edit.

[F. Refinement runs](F-refinement-runs.md) gives the run brief, the review checklist, and the failure modes.

### Steps for each item

1. Read the item and its comments. Then read the linked issues: parent, children, and blocked-by links.
2. Run the schema checks with `tbhbdev pm refine --check`, which reports each finding and each warning without a write.
3. Walk the applicable readiness checks in order and write one result for each, with evidence. A result is pass, fixed, or fail.
4. Decide the outcome from the results, using the table that follows.
5. Run the [Vale pass](#vale-pass) on the final body and on each comment the outcome posts, and fix each error before any write.
6. Write the outcome through `tbhbdev pm refine`, which posts the record and applies any links, closes, and edits in one call.

| Outcome | When it applies |
| --- | --- |
| `ready` | Each applicable check passes, either as written or after a fix that preserves the item's scope |
| `needs-info` | A fix would change intent, or a question needs an answer the run can't find |
| `split` | The work exceeds the size in item 6 and divides into pieces that each pass alone, or item 7 needs a Spike first |
| `drop` | Item 4 shows the work done or covered by a duplicate, or the reason for it has gone |

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

A fix may not change the meaning of the outcome sentence or the intent of a check. It may not change a Priority that someone already set either. Each of those changes makes the outcome Needs info, and so does a check whose observable result the text doesn't make plain.

### Outcome writes

Use the refine verb to post one record comment per item, including in a batch.

Within the record, each applicable readiness check has a result and evidence. Next come the outcome and the identity of the run. The last lines give the batch Chore, if any, and the final body's Secure Hash Algorithm (SHA) digest of 256 bits.

- **`ready`** writes the record. If checks needed fixes, the refiner writes the body and field edits first. The record lists each fix and the check it served. GitHub's edit history keeps the earlier text.
- **`needs-info`** writes the record, which cites the question comment. The sync job sets Status to `Needs info`. [Decision D8](decisions.md#d8-should-ordinary-refinement-send-questions-of-operator-intent-as-one-approval-request-per-batch) covers how the operator receives those questions.
- **`split`** creates each child through the kernel in the right repository and links it as a sub-issue. When the children are Features, it retypes the original to an Epic. It then writes the record on the original. Each child gets its own refinement and record.
- **`drop`** writes the record with the close reason and then closes the item. If merged work met the outcome, close the item as completed. Close a duplicate with the duplicate reason and a link. Use not planned if the reason for the item has gone.

A question comment is a comment from the refiner on the issue with one numbered question per gap. For each question, the refiner identifies the answerer by role or specifies the operator. To reply, the answerer posts on the issue with a link to the question comment and the question number. A reply counts if it comes from the designated answerer and cites the question. For a batch Request under decision D8, a reply from the `sm` run counts if it cites the operator's answer.

To return the issue to the queue, the answerer puts the resolved information in the issue body or uses `tbhbdev pm refine --return`. The question comment includes these instructions. Either action makes the record stale. The sync job shows Needs refinement. An answer in a comment alone leaves the item at Needs info.

When the issue returns to the queue, the next refiner reads the prior records and the question comment. For each resolved question, the new record marks the comment link and question number as answered and cites the reply or body edit. Later records repeat these entries. The refiner skips answered questions. For a gap that still needs an answer, the refiner refers to the existing question.

### Vale pass

Every issue body and comment goes through Vale before the kernel writes it, using the organization's [`.vale.ini`](https://github.com/tbhb-dev/rules/blob/main/.vale.ini) and [`.vale/`](https://github.com/tbhb-dev/rules/tree/main/.vale) styles at the rules repository's current commit. This design copies none of their rules.

The run lints the final body before writing it, and each comment it posts before posting, the record included. Vale reads each as Markdown, as in `vale --ext=.md`.

An error from the configuration blocks the write of a body or comment at the short threshold or longer. In a short text, the run reports each error as a warning instead. A warning never blocks, and the refiner fixes it or keeps it with a one-line reason in the record.

A body counts as short when its prose runs under half the total-word cap of its variant in [A](A-content-model.md), such as 100 words for a Feature or 40 for a service Request. Comments and bodies with no total-word cap use a 65-word threshold.

Why the threshold? Some error-level rules measure spread across a whole document, and a few sentences give them too little to measure.

Vale reports findings and leaves the text alone.

An error that blocks the write sends the refiner back to fix the text under the fix rules and run Vale again. If the only fix would change intent, the outcome becomes Needs info and the body keeps its earlier text. Every change to the text is a fix that the refiner makes by hand and the record lists, because the run doesn't apply a suggested replacement on its own.

Findings come back as a list of line, rule, level, and message, as in `12: Google.Semicolons (error) Use semicolons judiciously.` The record lists them under item 8.

If the Vale binary or the configuration is missing, the write fails closed. D10 in [decisions](decisions.md) asks the tools role to provide this pass through the shared content check.

### Evidence rules

Each claim in a record cites at least one source, in one of these forms:

- A commit appears as a full or 12-character SHA, with its repository.
- A pull request appears as `owner/repo#number`.
- A path appears at a commit, as `path@sha`.
- A command appears with the commit or query it ran against, and with its result count or a one-line result.

A pass on item 4 cites both a repo search and an issue search for work items and cites only an open duplicate search for a command Request. A record on a public item never cites a private repository. When the only evidence is private, the refiner writes it on the private twin.

The public record then says that one private citation exists. A review fails an unsupported claim, even one with a correct conclusion.

## Internal security topics

Every issue about internal security goes in a private repository. Internal security means a weakness in the organization's own systems, such as a vulnerability, a gap in access control, an exposed credential path, or a missing detection. A public advisory about an upstream dependency doesn't count.

If refinement finds such an item in a public repository, its first write transfers the item to the private twin. Where no twin exists, the item goes to the central work repository.

GitHub's transfer docs allow a move between repositories of one organization. They rule out only a move from a private repository to a public one.

The run writes the record and any fixes only after the move, so no record repeats the detail in public. A transfer hides the item from public view, but it can't recall copies made while the item was public. The record on the moved item says how long the item was public.

## Service requests

The shared refinement record belongs to the pm role. For Requests, the `sm` role maintains the mapping from checks to record fields. An Agent Request goes to an agent role. The operator handles command Requests and decision Requests. See the [Request kinds](../itsm-records/D-operator-requests.md#kinds-of-request) for the command and approval records. Apply check 5 to every Request variant.

| Item | Agent Request | Decision Request | Command Request |
| --- | --- | --- | --- |
| 1. Outcome | The ask says what someone can observe once it has an answer. | The Request says what changes for each answer. | The Changes section says what the script changes. |
| 2. Context | Applies. | Applies. | The reason an agent can't run the command makes sense on its own. |
| 3. Checks | Applies when the ask is a change. | Each option states an effect that someone can verify. | Checks first, Exit codes, Undo, and Dry run are each testable. |
| 4. Duplicates | Applies. | The search includes a past Decision on the same question. | The search checks only for an open duplicate Request. |
| 5. Fields | Apply item 5. | Check the `adr` link before closing. | Check the script hash. |
| 6. Size | Apply the size limit if the ask requires a change. | This check doesn't apply. | Apply the command Request check in item 6. The command also fits the pattern in the [command Request schema](../itsm-records/D-operator-requests.md#what-a-command-request-contains). |
| 7. Questions | Applies. | State the question for the operator and list the options. Recommend one. State what happens for each answer. | Applies. |
| 8. Sentences | Applies. | Applies. | This check applies to the prose. |
| 9. Approver | An agent approves the Request. The worker loads the subsystem context. | The operator decides. | The operator approves the command. |
| 10. Result check | Specify the evidence to check after the work. | Check the answer in the linked decision record. | Check the exit status and output tail on the Request. |

Check any parent and blocked-by links on a Request. An operator Request must explain why an agent account can't do the work. If the explanation is missing, fail item 5 and return the Request to its author.

For an operator Request, refinement runs while the Request is open, before the workflow moves it to waiting on operator. That workflow gains one condition for that move: a current ready record. The `sm` role makes that change in the service-management records design.

## Limits of refinement

- Refinement leaves existing Priority values in place. It follows the shared taxonomy when filling an empty Priority.
- It doesn't estimate effort, assign anyone, add an owner, or start work.
- Approving or denying a Request stays with the operator's request commands.
- Operator intent comes from the operator. Refinement asks.

## Changes to the rest of the design

- The verbs gain `refine` with an edit operation and a return operation. The variants gain the refinement record variant.
- The tools role validates labels and fields against the merged global and repository configuration for the issue type.
- The pm role adds an optional Context block of up to 60 words to the Task schema. The pm role uses the current Task template until the schema change merges.
- The [routing order](B-routing-and-taxonomy.md#routing-order-for-create) sends each internal-security item to a private repository.
- The kernel runs the Vale pass for every issue body and comment write. D10 proposes sharing the pass through the content check.
- The block model exposes the text of each paragraph and list item without its code spans, which the checks of D read.
- The service-management records design gains one condition before waiting on operator, as the service requests section describes.

## Sources

The web research pass of 2026-11-23 read these sources. Each line says what changed because of it, or what it confirmed.

- [The Scrum Guide (2020)](https://scrumguides.org/scrum-guide.html) has no DoR and treats refinement as ongoing. Added the Why a gate section and confirmed the size check of item 6.
- [Mountain Goat Software on the dangers of a DoR](https://www.mountaingoatsoftware.com/blog/the-dangers-of-a-definition-of-ready) warns that rules demanding finished upstream work form a stage gate. Added the limit that the DoR doesn't require a design, an estimate, or an assignee, and added the phase-gate failure mode in F.
- [Scrum.org on why the Scrum Guide omits the DoR](https://www.scrum.org/resources/blog/why-isnt-definition-ready-described-scrum-guide) makes the same point about overhead. Confirmed the phase-gate risk without a change.
- [Bill Wake's INVEST](https://xp123.com/articles/invest-in-good-stories-and-smart-tasks/) ties understanding a story to being able to test it. Confirmed items 3, 5, and 6 without a change.
- [The Gherkin reference](https://cucumber.io/docs/gherkin/reference/) says an outcome step checks what a user or outside system sees. Shaped items 1 and 3 so a check states what a person or another system can see.
- [GitHub's Copilot coding agent guidance](https://docs.github.com/en/copilot/tutorials/coding-agent/get-the-best-results) asks for a clear problem, acceptance criteria, and file pointers, and keeps exploratory work for people. Added the Spike split to item 7 and confirmed items 3 and 6.
- [Kubernetes issue triage](https://www.kubernetes.dev/docs/guide/issue-triage/) uses bot labels that change only by a command from a member, the author, or an assignee. It informed the use of authorized commands to change refinement state.
- [GitLab's handbook on how product work flows](https://handbook.gitlab.com/handbook/product-development/how-we-work/product-development-flow/) returns work to refinement when someone finds it incomplete after ready. Added the worker's return operation.
- [GitHub's issue form syntax](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-issue-forms) can set the type, labels, and Project. Each form checks only its own inputs. Added the paragraph on issues written outside the kernel.
- [GitHub's built-in Project workflows](https://docs.github.com/en/issues/planning-and-tracking-with-projects/automating-your-project/using-the-built-in-automations) include two that set Done and start enabled. Added the setup step that turns them off. Also added a failure mode in F and a change line.
- [GitHub's workflow token docs](https://docs.github.com/en/actions/concepts/security/github_token) say events the token causes don't start a new run. Added the scheduled run of the sync job.
- [GitHub's REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api) cap content creation at 80 a minute and 500 an hour. Replaced the unmeasured limit in E with these figures and added the per-minute budget and the backoff rule.
- [GitHub's sub-issue docs](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues) allow 100 sub-issues per parent. Added the line in E that the batch Chores fit under the import Epic.
- [GitHub's issue transfer docs](https://docs.github.com/en/issues/tracking-your-work-with-issues/administering-issues/transferring-an-issue-to-another-repository) allow a transfer within one owner and forbid a move from a private repository to a public one. Added the transfer of internal-security items.
- [GitHub's issue dependency docs](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/creating-issue-dependencies) show blocked-by links in Projects. Confirmed that links record relations, which backs rule 4 of D, without a change.
- [Google's style guide on lists](https://developers.google.com/style/lists) lets a fragment end without a period. Kept the stricter rule 3 of D, which doesn't allow fragments.
- [Digital.gov on short sentences](https://digital.gov/guides/plain-language/writing/clear-short) asks for one idea per sentence. Confirmed rule 2 of D without a change.
