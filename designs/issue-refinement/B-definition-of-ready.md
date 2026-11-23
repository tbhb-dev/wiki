---
type: Design Appendix
title: Definition of Ready
description: The checks that work items and service requests pass before work begins.
tags: [issues, refinement, ready, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Definition of ready

Back to [the design](README.md).

## Why a gate

Refinement gives the worker enough information to start and check the result. The Scrum Guide treats refinement as ongoing work. The readiness gate gives an agent that information before it starts work. Readiness can precede design and estimation. Assignment can wait too.

## Scope

The numbered checks apply to pm work items. Apply each check that fits the variant. Use the [service readiness checks](#service-requests-and-problems) for the records that the service-management role (`sm`) handles. Every issue follows the writing rules below, with the conflicts in existing service variants tracked in [decisions](decisions.md#d4-which-service-rules-conflict-with-the-shared-rules).

## Readiness checks

A Configure Unify Execute (CUE) schema checks the block model. The kernel checks rules that need live data. A repository read checks a cited commit. The refiner records the reason for each judgment.

1. **The Outcome states an observable end state.** Write one sentence about what a reader can observe after the work. A Defect uses its expected behavior, and a Spike identifies the decision its experiment unblocks. Apply the [observable results rule](#observable-results). The schema checks the sentence count. The refiner judges the result.
2. **The Context makes sense to a newcomer.** Identify the repository, component, and behavior the reader needs to understand. Explain unfamiliar terms. Keep retired notes, run names, process dates, raw identifiers, file-line coordinates, and mapping notes out. Keep a date only as part of a fixed fact, such as a decision date with its reference. Apply the [durable references rule](#durable-references) when a reference is necessary. The schema checks the mechanical forms. The refiner checks readability and relevance.
3. **Each check states a distinct observable result within the variant's caps.** Apply the [observable results rule](#observable-results). The schema checks the task list and the caps. The refiner judges whether a second person can confirm the result.
4. **The work remains undone and no other open item covers it.** Search the responsible repository at its current commit. Search open issues in that repository, in its private twin when the refiner has access, and in the central work repository. Cite each search and its result count in the record.
5. **Relations, org issue fields, and labels fit the variant.** Check the [routing order](../pm/B-routing-and-taxonomy.md#routing-order-for-create) and the [internal security rule](A-refinement-procedure.md#internal-security-topics). Check the parent type and any required parent. Check that each dependency resolves to an issue. The kernel checks live data against the taxonomy in `designs/issue-taxonomy/` and the merged global and repository configuration. The refiner checks the evidence for any value they fill in.
6. **One pull request can deliver the change under the size limits.** Compare the likely paths and size with the target repository's `pr_size` limits. An estimate over half either limit or work across repositories calls for a split. An Epic or Initiative passes when its children together cover its outcome. The refiner cites a repository read of the likely paths.
7. **The reader has the answers they need to start.** The schema checks for markers such as `TBD` and `TODO`. If the reader needs an answer before work begins, record `needs-info` with the question and its answerer. Split off a Spike when the work needs an experiment first. Apply the [hedge check](#unresolved-hedges) to factual claims.
8. **Each sentence and list item follows the writing rules.** Apply the [prose rules](#writing-rules), the mechanical [content checks](#content-checks), and the [Vale pass](A-refinement-procedure.md#vale-pass). The refiner checks clarity and comma splices.
9. **A Request identifies its handler and authority.** See the [Request workflow check](#requests).
10. **A Request specifies its answer path.** See the [Request workflow check](#requests).
11. **The title and Outcome name the change and its end state.** Apply the [title and process-verb rules](#titles-and-process-verbs). The refiner splits unrelated changes through the [split outcome](A-refinement-procedure.md#outcome-writes).
12. **Context gives current readers the facts they need.** Apply the [durable references rule](#durable-references). The refiner judges whether the reference adds information and still identifies the intended work.
13. **Decision work uses a Request.** File a choice that needs an answer as a Request, including a Chore titled `Decide X`. Apply the [Handler rule](#requests) to determine who answers.
14. **Resolve factual uncertainty.** Apply the [hedge check by type](#unresolved-hedges).
15. **Issue sections contain no procedure.** Apply the [no-procedure rule](#no-procedure) to every section.

The schema checks the draft at creation and on edits through the kernel. For issues created through the GitHub web interface, `tbhbdev pm refine --check` runs the same schema checks. See [Status transitions](D-status-transitions.md) for the sync behavior before verification.

## Writing rules

- Write one fact per sentence in plain subject-verb order. Use a separate sentence for the next fact.
- End each full-sentence list item with a period, including each item in a task list.
- Leave tracking, issue structure, and edit history to GitHub. Use issue relations for the hierarchy and dependencies.
- Use the responsible role kind's `role/` label from the [agent roles registry](../agent-roles/README.md). Keep owner lines and owner metadata out of pm issues.

### Observable results

The Outcome describes the state of the system after the work. Each check states one distinct result and identifies evidence a reader can inspect, such as the output of a named check or the content at a stable path. Where the variant has both Acceptance criteria and Verification, the two sections cover different conditions.

Skip bookkeeping criteria. An issue existing, a pull request merging, or a document changing tells the reader nothing about the result. Describe what the issue delivers. An observable result reads `The harvest log shows No harvests yet when it has no entries.` A Research issue's named report is the deliverable. See the [Research done check](../pm/A-content-model.md#research).

### Durable references

Use a fully qualified issue reference of the form `owner/repo#N` whenever the text refers to other work. Phrases such as `a separate issue`, `a follow-up`, and `another security debt` leave the work unidentified. Record related work that must finish first through a `blocked_by` relation. Leave dependency prose out.

Keep a necessary reference once. Use an issue reference where possible in place of a raw commit hash, run identifier, tracker identifier, or file-line coordinate. Put evidence that belongs to the refinement record in the [record's evidence](A-refinement-procedure.md#evidence-rules).

### Titles and process verbs

- Identify the thing and the change. Exclude vague objects such as `check errors`, `issues`, `problems`, `improvements`, `follow-ups`, and `debt`.
- Give each issue one change. Split a title that counts changes with words such as `two`, `three`, `several`, `various`, or `multiple`. Split unrelated changes joined by `and` or `or`.
- Put provenance in Source or Context. Leave phrases such as `found in ... reviews`, `from the audit`, `raised by`, and `noted during` out of the title.
- Start the title and Outcome with the change or the resulting state. Exclude process verbs such as `Resume`, `Continue`, `Finish`, `Keep`, `Restart`, `Retry`, `Revisit`, and `Follow up`.

The title `Fix two tools check errors found in github-metrics reviews` fails specificity, single-change scope, and provenance.

### No procedure

Keep undo, rollback, and how-to steps out of every issue section, including Context, Outcome, and Checks. Put the undo plan in the risk and rollback text of the pull request. Put standing instructions in a runbook. A check can state what to observe and where. In a Defect, use Reproduction for observed conditions and a reference to the runbook.

A failing Context for a fictional repository reads `Undo: delete the tunnel and remove the staged file.`

### Unresolved hedges

A hedge on a checkable fact blocks readiness. Look for `may already`, `might`, `possibly`, `probably`, `unclear whether`, `should be checked`, and `unknown whether`. Run a read-only check that settles the fact. State the result in the issue with its evidence. If the fact remains uncertain, record `needs-info`. Ask a specific question and identify the person or role who can answer. The work's own unknown can remain in the place the table gives it.

| Type | `Where an unknown may live` | Pass test |
| --- | --- | --- |
| Initiative | The variant lacks a scope section. See [decision D5](decisions.md#d5-where-do-open-scope-questions-go). | Resolve checkable facts before readiness. |
| Epic | A scope section needs a schema change. See [decision D5](decisions.md#d5-where-do-open-scope-questions-go). | The item answers factual questions that a read can settle. |
| Feature | Outside the issue until resolved. | State the intended behavior and settled facts. |
| Defect | The cause can remain unknown. | The issue states observed behavior as fact and identifies the cause as unknown when necessary. |
| Chore | Resolve factual uncertainty during refinement. | State the maintenance facts directly. |
| Task | Resolve unknown facts before readiness. | Give factual statements the worker can act on. |
| Spike | Question. | Give the experiment one explicit unknown and settle its factual premises. |
| Research | Question. | State what the report investigates once. Its premises pass the hedge check. |
| Problem | Question, proposed. The existing variant uses Cause. | State the investigation's unknown once. Cite the observations that support it. |

## Content checks

Text checks ignore case. For matching, the normalizer exposes prose from paragraphs and list items with code removed.

| Check | What it flags | Level |
| --- | --- | --- |
| Existence-only result | In Outcome or Checks, `is` or `has been` before `written`, `merged`, `updated`, `done`, `complete`, `completed`, or `finished`. A claim that a pull request merged also fails. | Refuse |
| Weak result | In Outcome or Checks, `exists`, `is added`, `is created`, `is implemented`, or `is documented`. A check that opens with an imperative such as `Write`, `Add`, or `Update` also warns. | Warn |
| Vague result | `works well`, `cleaner`, `improved`, `better`, or `faster` without a measure. | Warn |
| Semicolon | A semicolon in prose. | Refuse |
| Missing final period | A list item without a final period. | Refuse |
| Issue history | `tracked in`, `tracked under`, `this issue covers`, `this issue tracks`, `this issue is`, `see the sub-issues`, `split from`, `parent issue`, `child issue`, `umbrella`, or `this epic`. | Warn |
| Owner line | `Owner:`, `Owned by`, or an owner key in the metadata. | Refuse |
| Security topic in public | `vulnerability`, `exploit`, `CVE-`, `privilege escalation`, `auth bypass`, or `leaked credential`. | Warn |

In the issue schema, refusals are constraints and warnings are data beside the hints. Review each warning in context. A public dependency advisory can cite its Common Vulnerabilities and Exposures (CVE) number. For Research, the check can verify the report at the specified path. Before marking the issue ready, fix the text or record a reason to keep it.

A later pm schema change supplies the rules that the tools kernel runs. The title lint covers process verbs, number words, provenance phrases, quote marks, and vague category nouns used as objects. The kernel also checks hedge phrases and can flag vague references to other work. Judgment still covers intended scope, investigative questions, and whether the reported result is observable.

## Service requests and problems

### Requests

A Request becomes ready when the workflow passes its content check. The base variant uses the four sections below in order.

| Section | Check |
| --- | --- |
| Ask | Use 3 to 40 words. |
| Recommendation | Use 3 to 40 words. |
| Decisions checked | Use one Decision link per item. Each link must resolve. Use at most 40 words across those items. |
| Answer path | State where the answer goes in 1 to 25 words. |

Use at most 110 words for the base body and at most 70 characters for the title. Omit a trailing period in the title. Set Priority and Handler. For Handler Agent, use one area label. Use Handler Operator only for work that requires authority or access unavailable to agents. Give the reason, such as a credential or host agents lack access to, or a one-way door that requires operator authority.

The workflow moves `open` to `waiting on operator` when the check passes. An operator question moves it to `waiting on author`. During refinement, any author edit returns it to `open` for another check.

Kind `command` uses the command sections Changes, Checks first, Exit codes, Undo, Dry run, Why not an agent, and Command. Its variant caps the body at 80 words. Kind `change` is a Request for Change (RFC). Its variant has What, Why, Risk, and Approver and authority sections and requires a Class of `normal` or `emergency`. On approval, the verb opens a Change and links the Change and the Request in both directions. These are the service workflow's current shapes. See [operator requests](../itsm-records/D-operator-requests.md#kinds-of-request) and the [recorded conflicts](decisions.md#d4-which-service-rules-conflict-with-the-shared-rules).

### Problems

A Problem is ready for investigation when it meets four conditions.

- Scope states the symptoms and the hosts or repositories in scope in one sentence.
- Evidence includes at least one dated Incident, occurrence, or failing check.
- Owner is one role label from the live list.
- Next step names one action with a time box. A step too large for one action becomes a Spike sub-issue.

Keep the staged item as a Defect if the work fits that type by itself. If it belongs to an existing Problem, make it a child of that Problem. Otherwise, open a new Problem. Link similar items as Related until evidence shows one cause. A verb sets Priority after refinement. See [Problem refinement](../itsm-records/problems.md#refinement).

### Records opened complete

A verb opens each Incident and Change complete. Neither type has a refinement step.
