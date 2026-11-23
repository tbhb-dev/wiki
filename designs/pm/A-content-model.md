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
| `taxonomy.cue` | Priority, Work type, Severity, Handler, area and role label patterns, time box, and the issue reference pattern |
| `issue.cue` | The shared issue section, title, and item rules |
| `issue.<variant>.cue` | One variant with its sections in order, caps, metadata, and hints |

A run loads core, taxonomy, the issue base, and exactly one variant, because each variant file defines the same `#Variant` name. The `#Hints` definition keys its hints by variant.

The pm variant schemas use `#WorkType` from the [shared taxonomy](https://github.com/tbhb-dev/tools/blob/main/schemas/taxonomy.cue). Its values are `Planned` and `Unplanned`. The destination variants already accept an optional `source` field. The import sets it to `Import` at creation. See [E. Tracker import](E-import.md#staging-and-refinement).

The `schemas/issue.imported.cue` variant file stays in the pm repository until the import finishes. The pm role then removes the file.

## Metric semantics

Real output from the normalizer showed these rules (proven):

- A section's own level-two heading counts once in its heading metrics, so the shared section rule sets exactly one number-sign heading. The heading's words don't enter the section's word count.
- A list item contains a paragraph, so a section made of a list has as many paragraphs as items. The section rules don't pin paragraphs on list sections.
- A section's content holds blank-line blocks, paragraphs, and list blocks. It excludes the heading.
- Draft totals sum the section bodies.
- Source characters stop at eight times the words for a section and ten times for the draft, which keeps hidden text such as comments and long URLs from bypassing a word cap.

## The shared issue section

The shared rule bars tables, HTML blocks, block quotes, extra headings, thematic breaks, and link reference definitions from every issue section. It allows a list depth of one. A title runs 15 to 70 characters with at least three words, and it can't end in a period, an exclamation mark, or a colon. A Spike question must end in a question mark.

## The variants

| Variant | Total words | Notes |
| --- | --- | --- |
| Feature | 200 | Outcome, Checks as a task list, and optional Context and Out of scope |
| Chore | 150 | Parent required, because tasks are sub-issues |
| Defect | 220 | Actual and Reproduction required, severity required |
| Spike | 130 | Question, Decision it unblocks, Checks as deliverables, and a time box |
| Epic | 180 | Parent optional, so an Epic can belong to an Initiative |
| Initiative | 180 | No parent |
| Service Request | 80 | Body prose cap, based on the [operator command Request](../itsm-records/D-operator-requests.md#what-a-command-request-contains) |

Floors are low or absent. A forced section produces filler text, so only sections that give the type its meaning need text.

A Task always has a parent. It requires a Verification section. Its optional Context section has a limit of 60 words.

## Refusals and the guide

`cue vet --all-errors` returns every finding in one pass. The formatter maps each finding's path to a section, a rule, a limit, the actual value, and the hint. When any other finding exists, it drops the block-order finding, because a failing block disappears from the derived order and adds noise. The guide generator reads the same schema, so the two can't disagree. Both are new and unverified until built.

## Checks outside the schema

The kernel keeps a named list of checks that need live data, issue relations, or text rules that CUE can't express, as [decision D4](decisions.md#d4-how-do-the-writing-rules-stay-in-one-place) allows. The list covers links-only sections and note flooding. The kernel checks that a cited Request has approval and that a required retrospective request for comments exists. It also checks parent types and validates area and role labels against the live lists. A Defect's parent is optional under [readiness check 5](C-refinement.md#readiness-checks). If it has a parent, the type must be Problem or Epic.
