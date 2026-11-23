---
type: Design Appendix
title: Writing rules for project management issues
description: The writing rules every issue follows, with good and bad pairs, the content checks that flag a break, and how the templates, the guide, and the import apply them.
tags: [pm, writing, content, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Writing rules for project management issues

Back to [the design](README.md).

These rules apply to every work item variant and every service request. Refinement checks them as items 1, 3, and 8 of the Definition of Ready (DoR) in [C. Refinement](C-refinement.md). Everything here is new.

## The rules

1. **Each outcome and check states an observable result.** A person or another system can see the result and agree on it. It never says only that the work exists or that someone did it.
2. **Each sentence states one fact.** It reads as a plain subject-verb sentence that a stranger takes in once. A semicolon or a comma splice never joins two facts, and separate facts become separate sentences or bullets.
3. **Each list item is a full sentence that ends with a period.** Task-list checks follow the same rule. No list item is a fragment.
4. **No sentence describes the issue itself.** Sentences about the issue's structure, tracking, relations, or history don't belong. GitHub's sub-issue and dependency links record those. Each sentence says something about the work or the system.

## Good and bad pairs

The bad examples appear in code spans, which keeps this page's own prose checks off them. A bad meta sentence often has no rewrite, and then the fix deletes it.

| Rule | Bad | Good |
| --- | --- | --- |
| 1 | `The runbook is updated.` | Following the runbook on a fresh host brings the service up, and its health endpoint returns 200. |
| 1 | `The spike is written.` | The spike's notes give a yes or no for each of the three storage backends, with the command that showed it. |
| 1 | `The migration pull request is merged.` | Running `tbhbdev pm guide defect` prints the Severity section with its hint. |
| 2 | `The sync job reads labels, so a hand-added label does nothing; only the record counts.` | The sync job ignores a label that has no current record. |
| 2 | `The cache is per host, it expires after an hour.` | The cache holds entries for one host. Each entry expires after an hour. |
| 3 | `- Retry on 429` | `- The client retries after a 429 response.` |
| 3 | `- [ ] Health endpoint` | `- [ ] The health endpoint returns 200 within 2 seconds of start.` |
| 4 | `Its parts live under the gateway rollout epic.` | Delete it. The sub-issue link already shows the parent. |
| 4 | `This issue covers the cache work split from the storage epic.` | A repeated read returns from the cache without a call to the storage backend. |
| 4 | `See the sub-issues for details.` | Delete it. GitHub lists the sub-issues under the body. |

Rule 1 rewrites an existence claim into the result the work makes visible. Ask what someone would run or look at to know the work succeeded, and write that down.

## Content checks

Matching is case-insensitive and reads the text of every paragraph and list item without code spans or code blocks, which the block model doesn't expose yet. The normalizer gains that text before any of these checks can run.

| Check | What it flags | Level |
| --- | --- | --- |
| Existence-only result | In an outcome or a check: `is` or `has been` followed by `written`, `merged`, `updated`, `done`, `complete`, `completed`, or `finished`, and a pull request named as merged | Refuse |
| Weak existence result | In an outcome or a check: `exists`, `is added`, `is created`, `is implemented`, or `is documented`, and a check that opens with an imperative verb such as Write, Add, or Update | Warn |
| Vague result | Works well, cleaner, improved, better, and faster without a number | Warn |
| Semicolon | A semicolon in prose | Refuse |
| Missing final period | A list item whose text doesn't end with a period | Refuse |
| Meta exposition | `tracked under`, `this issue covers`, `this issue tracks`, `this issue is`, and `see the sub-issues` | Refuse |
| Possible meta exposition | `tracked in`, `split from`, `parent issue`, `child issue`, `umbrella`, and `this epic` | Warn |

A comma splice has no reliable pattern. The refiner judges it under item 8. Each warn phrase can be legitimate. An issue about the pm tools may need to say parent issue, and a Helm chart can be an umbrella chart.

Each refuse check becomes a constraint in the variant schema, which keeps the rule in one home as D4 requires. A warn check is part of the schema too, as data beside the hints. The kernel refuses no write for a warning, and `tbhbdev pm refine --check` lists each match for the refiner to clear with a written reason.

The imported variant reports every check here as a warning. Refinement clears each one when it converts the item into its real variant.

## Templates and the guide

- Each template's sample text follows the four rules and passes each check of its variant.
- Each sample list item in a template is a full sentence that ends with a period.
- `tbhbdev pm guide <type>` prints the four rules with one good and bad pair each, from hint data in the schema.
- A refusal from one of these checks gives the rule as its hint, with the good example of its pair.

## Imports

The import in [E. Tracker import](E-import.md) maps each original issue into the Context section of the imported variant. While it maps, it strips each sentence that describes the issue itself, such as a note on which tool tracks the work or which issue it came from. Where the earlier tracker recorded that relation as data, the import writes it as a sub-issue or dependency link instead.
