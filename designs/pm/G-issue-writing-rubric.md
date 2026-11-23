---
type: Design Appendix
title: Issue writing rubric
description: Pass and fail tests for reviewing the writing of each issue.
tags: [pm, writing, refinement, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Issue writing rubric

Back to [the design](README.md).

A judge model or a human reviewer applies one test per rule to each item separately. Use the full rules in [D. Issue writing](D-issue-writing.md) and [C. Refinement](C-refinement.md).

## Tests

| Rule | Applies to | Pass test | Fail test |
| --- | --- | --- | --- |
| R1. Outcome states an end state. See [writing rule 1](D-issue-writing.md#the-rules) and [readiness check 1](C-refinement.md#readiness-checks). | Outcome | The sentence says in plain words what someone can observe about the system after the work. | The sentence says only that the work exists or describes an action someone takes, such as `Add`, `Fix`, `Run`, or `Update`. |
| R2. Context makes sense to a newcomer. See [readiness check 2](C-refinement.md#readiness-checks). | Context | A newcomer with no tracker history can follow the text. | A sentence uses an unexplained internal name, nickname, or abbreviation or relies on a tracker row or earlier discussion for its explanation. |
| R3. Context omits raw identifiers. See [readiness check 2](C-refinement.md#readiness-checks). | Context | The text omits commit hashes, run and tracker identifiers, and file-line coordinates except for one fully qualified reference the reader needs. | A raw hash or identifier appears, or a reference repeats a link or falls outside the exception. |
| R4. Context omits dated claims. See [readiness check 2](C-refinement.md#readiness-checks). | Context | Each statement is true without a date. | A statement depends on a date or a qualifier such as `currently`, `now`, or `today`. |
| R5. Context states facts only. | Context | Each sentence states a fact. | A sentence gives an instruction or a procedure. |
| R6. Checks state distinct results. See [readiness check 3](C-refinement.md#readiness-checks) and [the rule on observable results](D-issue-writing.md#the-rules). | Checks and Outcome | Each check states an observable result that differs from the Outcome and every other check. | A check describes an action such as `Run`, `Add`, or `Update` or repeats a result from the Outcome or another check. |
| R7. Acceptance criteria and Verification cover different conditions. | Acceptance criteria and Verification | The sections state different conditions, or the type uses just one of the sections. | Both sections state the same condition. |
| R8. Title names the thing and the change. | Title | A reader can tell from the title alone what changes and where. | The title leaves the change or its target unclear, as with a category alone such as `check errors`, `issues`, `problems`, `improvements`, `follow-ups`, or `debt`. |
| R9. Title specifies one change. See [readiness check 6](C-refinement.md#readiness-checks) for scope and [the split outcome](C-refinement.md#steps-for-each-item). | Title | The title describes one change. | The title counts changes with a word such as `two`, `three`, `several`, `various`, or `multiple`, or joins unrelated changes with `and` or `or`. |
| R10. Title omits provenance. See [writing rule 4](D-issue-writing.md#the-rules). | Title | The title omits how someone found the work. | The title says where the work came from with a phrase such as `found in`, `from the audit`, `raised by`, or `noted during`. |
| R11. Title and Outcome omit process verbs. | Title and Outcome | Neither starts with `Resume`, `Continue`, `Finish`, `Keep`, `Restart`, `Retry`, `Revisit`, or `Follow up`. | Either starts with one of those verbs. |
| R12. Type fits the work. | Issue type | The work fits the type definition, which for a Chore requires a maintenance end state with no user-visible change. | The work falls outside its type definition, as when a Chore resumes, monitors, or operates a process. |

For R9, split the item if the title describes more than one change. For R10, put provenance in Source or explain it in Context when the reader needs it. For R11, use a Request for `Decide X`. See [readiness check 7](C-refinement.md#readiness-checks).

## Worked examples

These examples use a fictional repository called `thimble-orchard`.

| Rule | Failing text | Passing rewrite |
| --- | --- | --- |
| R1 | `Add an empty state to the harvest log.` | The harvest log shows `No harvests yet` when it has no entries. |
| R2 | `The basket counter uses the RHM logic from the tracker.` | The basket counter includes baskets that staff have marked as damaged. |
| R9 | `Show harvest dates in the log and print basket weights on labels` | Split the item into `Show the harvest date in the log` and `Print the basket weight on each label`. |

## Verdict format

For each item, the judge returns `pass`, `fail`, or `not applicable` for each rule. Each `fail` includes one sentence of evidence. A rewrite passes only when every applicable rule passes.

The rubric is advisory until the kernel runs the lint. That lint covers only the mechanical subset (process verbs, number words, provenance phrases, quote marks, and vague category nouns).
