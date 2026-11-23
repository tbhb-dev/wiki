---
type: Design Appendix
title: Project management content model
description: The block model that the schemas check, what each metric counts, and the reasons for the schema structure.
tags: [pm, content, schema, design]
status: draft
parent: README.md
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:5b8e83e63b1f7072
    at: "2026-11-22T21:39:34Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Project management content model

Back to [the design](README.md).

This appendix gives the reasoning to read beside the `.cue` files in the pm repository. The Configure Unify Execute (CUE) language defines them. Each claim says whether a test run proved it, current code shows it, or nobody has checked it yet.

## The block model

A check on raw Markdown counts words and lines. A table, an HTML block, or a long link can avoid a word cap. The kernel first normalizes the text into the GitHub-flavored block model (CommonMark plus tables and task lists), and the schemas constrain that model. Proven on the normalizer: seven deliberate mutations of a feature issue each fail with the expected finding.

## The files

| File | Content |
| --- | --- |
| `core.cue` | The draft, block, inline, and metrics definitions that every object shares |
| `taxonomy.cue` | Shared taxonomy rules, the time box, and the issue reference pattern |
| `issue.cue` | The shared issue section, title, and item rules |
| `issue.<variant>.cue` | One variant with its sections in order, caps, metadata, and hints |

A run loads core, taxonomy, the issue base, and exactly one variant, because each variant file defines the same `#Variant` name. The `#Hints` definition keys its hints by variant.

The `schemas/issue.imported.cue` variant file stays in the pm repository until the import finishes. The pm role then removes the file.

## Metric semantics

Real output from the normalizer showed these rules (proven):

- A section's own level-two heading counts once in its heading metrics, so the shared section rule sets exactly one number-sign heading. The heading's words don't enter the section's word count.
- A list item contains a paragraph, so a section made of a list has as many paragraphs as items. The section rules don't pin paragraphs on list sections.
- A section's content holds blank-line blocks, paragraphs, and list blocks. It excludes the heading.
- Draft totals sum the section bodies.
- Source characters stop at eight times the words for a section and ten times for the draft, which keeps hidden text such as comments and long URLs from bypassing a word cap.

## The shared issue section

The shared rule bars tables, HTML blocks, block quotes, extra headings, thematic breaks, and link reference definitions from every issue section. It allows a list depth of one. A title runs 15 to 70 characters with at least three words, and it can't end in a period, an exclamation mark, or a colon. End a Spike question with a question mark.

## The variants

The tools role defines the issue types, field and label values, and type-by-field matrix in `designs/issue-taxonomy/`.

| Variant | Total words | Notes |
| --- | --- | --- |
| Feature | 200 | Outcome, Acceptance criteria as a task list, and optional Context and Out of scope |
| Chore | 150 | Maintenance with an end state and no user-visible change. Work that resumes, monitors, or operates a process falls outside this variant. A Chore requires a parent. |
| Defect | 220 | Actual and Reproduction required |
| Spike | 130 | A Spike is a time-boxed experiment, often a throwaway build, that answers a question to unblock a decision. The experiment yields an answer, while Research requires a written finding. Its sections include Question, Decision it unblocks, and Acceptance criteria. The Acceptance criteria list the deliverables. The shared taxonomy defines its time box. |
| Research | 150 | See [Research](#research) for the proposed variant. |
| Epic | 180 | Parent optional, so an Epic can belong to an Initiative |
| Initiative | 180 | No parent |
| Service Request | By variant | See the [Request workflow check](../issue-refinement/B-definition-of-ready.md#requests). |

Floors are low or absent. A forced section produces filler text, so only sections that give the type its meaning need text.

A Task always has a parent. It requires a Verification section. A Task may include a Context section of up to 60 words once the pm role changes the Task schema (proposed). Until then, the Task schema refuses a Context section.

### Research

Research is an investigation that delivers a written finding as a report in the research repository or its private twin. A Spike uses a time-boxed experiment to unblock a decision, while the Research deliverable is the report.

Write the six required sections in the order shown. Use at most 150 visible body words across all six sections.

| Section | Word cap | What it states |
| --- | --- | --- |
| Question | 20 | State one question in one sentence ending in a question mark. |
| Decision or work it informs | 25 | Identify the decision or work the finding informs. |
| Scope | 45 | State what the investigation covers and what it leaves out. |
| Deliverable location | 35 | Identify the research repository by role and give the report's path pattern, without linking to a private repository. |
| Time box | 10 | Choose a time box from the same set that Spike uses. |
| Verification | 15 | Identify the report in one task-list item that verifies it exists at the specified path. |

Set Priority and Source. The sync job sets Status. Omit Work type, Severity, and Handler. A Research issue can have an Epic or Initiative as its parent, or it can have no parent. Exclude Tasks and Chores as parents.

After this design merges, the administrator creates the org issue type. In a later change, pm adds `issue.research.cue` using these sections and fields.

## Refusals and the guide

`cue vet --all-errors` returns every finding in one pass. The formatter maps each finding's path to a section, a rule, a limit, the actual value, and the hint. When any other finding exists, it drops the block-order finding, because a failing block disappears from the derived order and adds noise. The guide generator reads the same schema, so the two can't disagree. Both are new and unverified until built.

## Checks outside the schema

The kernel keeps a named list of checks that need live data, issue relations, or text rules that CUE can't express, as [decision D4](decisions.md#d4-how-do-the-writing-rules-stay-in-one-place) allows. Named checks cover links-only sections and note flooding. The kernel checks that a cited Request has approval and that a required retrospective request for comments exists. Outside CUE, the kernel also checks parent types. For fields and labels, it uses the merged global and repository configuration. Under [readiness check 5](../issue-refinement/B-definition-of-ready.md#readiness-checks), a Defect can have a parent. If it does, the type must be Problem or Epic.
