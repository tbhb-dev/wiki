---
type: Design Appendix
title: Refinement procedure
description: How a refinement run checks an item and records its result.
tags: [issues, refinement, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Refinement procedure

Back to [the design](README.md).

## Runs and inputs

A launching run starts a headless agent to refine one work item or a batch. The pm role refines work items and the service-management role (`sm`) refines Problems. GitHub stores refinement state, including edits, links, and records. See [service requests and problems](B-definition-of-ready.md#service-requests-and-problems) for the workflow check on Requests and the record types that skip refinement.

Assign a refiner who didn't write the body. Keep the reviewer independent of the refiner. Choose a reviewer who did no refinement work on the batch.

Record batch results in the Chore. On resumption, retry each pending close for an open item with a current `drop` record. Skip other current records. A follow-up run can revisit a split parent once the children together cover its outcome.

Its inputs are the issue list or query, the commit of each target repository, and the merged global and repository configuration for labels and issue fields. It also receives the variant guides from `tbhbdev pm guide`. Every write goes through the kernel, so the secret scan and the private-detail routing apply to each comment and edit.

[pm refinement runs](../pm/F-refinement-runs.md) gives the run brief, the review checklist, and the failure modes.

## Steps for each item

1. Read the item, its comments, and the issues linked as parents, children, or dependencies.
2. Run the schema checks with `tbhbdev pm refine --check`, which reports each finding and each warning without a write.
3. Walk the applicable [readiness checks](B-definition-of-ready.md#readiness-checks) in order and write one result for each, with evidence. Record a pass, a fix, or a failure.
4. Decide the outcome from the results, using the table that follows.
5. Run the [Vale pass](#vale-pass) on the final body and on each comment the outcome posts, and fix each error before any write.
6. Write the outcome through `tbhbdev pm refine`, which posts the record and applies any links, closes, and edits in one call.

| Outcome | When it applies |
| --- | --- |
| `ready` | Each applicable check passes, either as written or after a fix that preserves the item's scope |
| `needs-info` | A fix would change intent, or a question needs an answer the run can't find |
| `split` | The work splits under readiness check 6 or the title rule in check 11, or needs a Spike under check 7. |
| `drop` | Item 4 shows the work done or covered by a duplicate, or the reason for it has gone |

The refiner can make these fixes.

- Restate, trim, or move text between sections.
- Break a sentence into one fact per sentence, or into bullets.
- End each list item with a period.
- Delete a sentence that describes the issue itself.
- Rewrite a check that says only that someone did the work into the observable result the item's text already implies.
- Remove stale context.
- Fill an empty field from evidence.
- Correct a link.
- Transfer an internal-security item from a public repository to a private one, before any other write.

A fix preserves the meaning of the outcome, the intent of each check, and any existing Priority. Record `needs-info` when a fix would change any of those or the text leaves a check's intended result unclear. When the refiner fills an empty Priority on a pm item, use the shared taxonomy. The Problem path sets Priority [after refinement](B-definition-of-ready.md#problems). Refinement leaves estimates, assignments, and the start of work to the roles that do the work. The operator answers questions of operator intent.

## Outcome writes

Use the refine verb to post one record comment per item, including in a batch.

Within the record, each applicable readiness check has a result and evidence. Next come the outcome and the identity of the run. The last lines give the batch Chore, if any, and the final body's Secure Hash Algorithm (SHA) digest of 256 bits.

- **`ready`** writes the record. If checks needed fixes, the refiner writes the body and field edits first. The record lists each fix and the check it served. GitHub's edit history keeps the earlier text.
- **`needs-info`** writes the record, which cites the question comment. The [sync job](D-status-transitions.md) sets Status to `Needs info`. [pm decision D8](../pm/decisions.md#d8-should-ordinary-refinement-send-questions-of-operator-intent-as-one-approval-request-per-batch) covers how the operator receives those questions.
- **`split`** creates each child through the kernel in the right repository and links it as a sub-issue. When the children are Features, it retypes the original to an Epic. It then writes the record on the original. Each child gets its own refinement and record.
- **`drop`** writes the record with the close reason and then closes the item. If merged work met the outcome, close the item as completed. Close a duplicate with the duplicate reason and a link. Use not planned if the reason for the item has gone.

A question comment is a comment from the refiner on the issue with one numbered question per gap. For each question, the refiner identifies the answerer by role or specifies the operator. To reply, the answerer posts on the issue with a link to the question comment and the question number. A reply counts if it comes from the designated answerer and cites the question. For a batch Request under pm decision D8, a reply from the `sm` run counts if it cites the operator's answer.

To return the issue to the queue, the answerer puts the resolved information in the issue body or uses `tbhbdev pm refine --return`. The question comment includes these instructions. Either action makes the record stale. The sync job shows Needs refinement. An answer in a comment alone leaves the item at Needs info.

When the issue returns to the queue, the next refiner reads the prior records and the question comment. For each resolved question, the new record marks the comment link and question number as answered and cites the reply or body edit. Later records repeat these entries. The refiner skips answered questions. For a gap that still needs an answer, the refiner refers to the existing question.

## Vale pass

Before each write, the kernel runs Vale with the organization's [`.vale.ini`](https://raw.githubusercontent.com/tbhb-dev/rules/main/.vale.ini) and [`.vale/`](https://github.com/tbhb-dev/rules/tree/main/.vale) styles at the rules repository's current commit. Run `vale --ext=.md` on the final body and each comment, including the record. An error from the configuration blocks the write of a body or comment at the short threshold or longer. In a short text, the run reports each error as a warning instead. A warning never blocks, and the refiner fixes it or keeps it with a one-line reason in the record.

A body counts as short when its prose runs under half the total-word cap of its variant in [the pm content model](../pm/A-content-model.md#the-variants) or the applicable service record variant. Comments and bodies with no total-word cap use a 65-word threshold.

Vale reports findings without editing the text. An error that blocks the write sends the refiner back to fix the text under the fix rules and run Vale again. If the only fix would change intent, record `needs-info` and keep the earlier text. The refiner makes each fix by hand and lists it in the record. Findings come back as a list of line, rule, level, and message, as in `12: Google.Semicolons (error) Use semicolons judiciously.` The record lists them under item 8.

The kernel refuses writes if Vale or its configuration is missing. [pm decision D10](../pm/decisions.md#d10-can-the-content-check-use-vale-before-each-body-or-comment-write-and-report-only-warnings-for-short-text) asks the tools role to provide this pass through the shared content check.

## Evidence rules

Cite evidence for each claim in the record.

- A commit appears as a full or 12-character SHA, with its repository.
- A pull request appears as `owner/repo#number`.
- A path appears at a commit, as `path@sha`.
- A command appears with the commit or query it ran against, and with its result count or a one-line result.

For check 4, cite the searches and their results. Keep private repository citations out of public records. When only private evidence supports a claim about a public item, write the citation on the private twin and acknowledge it on the public item. The reviewer fails a claim that lacks evidence.

## Internal security topics

Every issue about internal security goes in a private repository. Internal security means a weakness in the organization's own systems, such as a vulnerability, a gap in access control, an exposed credential path, or a missing detection. A public advisory about an upstream dependency doesn't count.

If refinement finds such an item in a public repository, its first write transfers the item to the private twin that the [routing order](../pm/B-routing-and-taxonomy.md#routing-order-for-create) picks.

The run writes the record and any fixes only after the move, so no record repeats the detail in public. A transfer hides the item from public view, but it can't recall copies made while the item was public. The record on the moved item says how long the item was public.

## Sources

The web research pass of 2026-11-23 read these sources. Each line says what changed because of it, or what it confirmed.

- [The Scrum Guide (2020)](https://scrumguides.org/scrum-guide.html) has no DoR and treats refinement as ongoing. Added the [gate rationale](B-definition-of-ready.md#why-a-gate) and confirmed the size check of item 6.
- [Mountain Goat Software on the dangers of a DoR](https://www.mountaingoatsoftware.com/blog/the-dangers-of-a-definition-of-ready) warns that rules demanding finished upstream work form a stage gate. Added the limit that the DoR doesn't require a design, an estimate, or an assignee, and added the phase-gate failure mode in [pm refinement runs](../pm/F-refinement-runs.md#failure-modes).
- [Scrum.org on why the Scrum Guide omits the DoR](https://www.scrum.org/resources/blog/why-isnt-definition-ready-described-scrum-guide) makes the same point about overhead. Confirmed the phase-gate risk without a change.
- [Bill Wake's INVEST](https://xp123.com/articles/invest-in-good-stories-and-smart-tasks/) ties understanding a story to being able to test it. Confirmed items 3, 5, and 6 without a change.
- [The Gherkin reference](https://cucumber.io/docs/gherkin/reference/) says an outcome step checks what a user or outside system sees. Shaped items 1 and 3 so a check states what a person or another system can see.
- [GitHub's Copilot coding agent guidance](https://docs.github.com/en/copilot/tutorials/coding-agent/get-the-best-results) asks for a clear problem, acceptance criteria, and file pointers, and keeps exploratory work for people. Added the Spike split to item 7 and confirmed items 3 and 6.
- [Kubernetes issue triage](https://www.kubernetes.dev/docs/guide/issue-triage/) uses bot labels that change only by a command from a member, the author, or an assignee. It informed the use of authorized commands to change refinement state.
- [GitLab's handbook on how product work flows](https://handbook.gitlab.com/handbook/product-development/how-we-work/product-development-flow/) returns work to refinement when someone finds it incomplete after ready. Added the worker's return operation.
- [GitHub's issue form syntax](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-issue-forms) can set the type, labels, and Project. Each form checks only its own inputs. Added the paragraph on issues written outside the kernel.
- [GitHub's built-in Project workflows](https://docs.github.com/en/issues/planning-and-tracking-with-projects/automating-your-project/using-the-built-in-automations) include two that set Done and start enabled. Added the administrator's [setup step](D-status-transitions.md#sync-triggers-and-setup) and the corresponding [failure mode](../pm/F-refinement-runs.md#failure-modes).
- [GitHub's workflow token docs](https://docs.github.com/en/actions/concepts/security/github_token) say events the token causes don't start a new run. Added the scheduled run of the sync job.
- [GitHub's REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api) cap content creation at 80 a minute and 500 an hour. Replaced the unmeasured limit in [the tracker import](../pm/E-import.md#throttling) with these figures and added the per-minute budget and the backoff rule.
- [GitHub's sub-issue docs](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues) allow 100 sub-issues per parent. Added the line in [the tracker import](../pm/E-import.md#order-and-batches) that the batch Chores fit under the import Epic.
- [GitHub's issue transfer docs](https://docs.github.com/en/issues/tracking-your-work-with-issues/administering-issues/transferring-an-issue-to-another-repository) allow a transfer within one owner and forbid a move from a private repository to a public one. Added the transfer of internal-security items.
- [GitHub's issue dependency docs](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/creating-issue-dependencies) show blocked-by links in Projects. Confirmed that links record relations, which backs the [issue-history rule](B-definition-of-ready.md#writing-rules), without a change.
- [Google's style guide on lists](https://developers.google.com/style/lists) lets a fragment end without a period. Kept the [full-sentence rule](B-definition-of-ready.md#writing-rules).
- [Digital.gov on short sentences](https://digital.gov/guides/plain-language/writing/clear-short) asks for one idea per sentence. Confirmed the [one-fact rule](B-definition-of-ready.md#writing-rules).
