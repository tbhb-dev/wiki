---
type: Design Appendix
title: Project management content model
description: The block model that the schemas check, what each metric counts, and the reasons for the schema structure.
tags: [pm, content, schema, design]
status: draft
parent: README.md
---

# Project management content model

Back to [the design](README.md).

This appendix gives the reasoning to read beside the `.cue` files in the pm repository. The Configure Unify Execute (CUE) language defines them. Each claim says whether a test run proved it, current code shows it, or nobody has checked it yet.

## The block model

A check on raw Markdown counts words and lines. A table, an HTML block, or a long link can avoid a word cap. The kernel first normalizes the text into the GitHub-flavored block model (CommonMark plus tables and task lists), and the schemas constrain that model. Proven on the normalizer: seven deliberate mutations of a feature issue each fail with the expected finding.

## The files

| File | Content |
| --- | --- |
| `core.cue` | The draft, block, inline, and metrics definitions that every object shares |
| `taxonomy.cue` | Priority, work type, severity, handler, area and role label patterns, time box, and the issue reference pattern |
| `issue.cue` | The shared issue section, title, and item rules |
| `issue.<variant>.cue` | One variant with its sections in order, caps, metadata, and hints |

A run loads core, taxonomy, the issue base, and exactly one variant, because each variant file defines the same `#Variant` name. The `#Hints` definition keys its hints by variant.

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

Floors are low or absent. A forced section produces filler text, so only sections that give the type its meaning need text.

## Refusals and the guide

`cue vet --all-errors` returns every finding in one pass. The formatter maps each finding's path to a section, a rule, a limit, the actual value, and the hint. When any other finding exists, it drops the block-order finding, because a failing block disappears from the derived order and adds noise. The guide generator reads the same schema, so the two can't disagree. Both are new and unverified until built.

## Checks outside the schema

CUE can't express these, so they stay in the kernel. This list keeps them from becoming a second home for rules. They cover the parent's type (a Feature under an Epic), links-only sections, note flooding, a retrospective required at close, and the area and role label spellings against the live lists.
