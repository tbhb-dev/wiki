---
type: Design Draft
title: Project management for tbhb-dev
description: How issues, org issue fields, and a checked content kernel give the org a backlog that agents and people write the same way.
tags: [pm, issues, projects, content]
status: draft
owner: pm
updated: "2026-11-23"
decisions_open: 1
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

# Project management for tbhb-dev

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | pm | 2026-11-23 | One numbered decision open in [decisions.md](decisions.md). |

## Summary

Each work item starts as an issue in the GitHub repository responsible for the work. The `work` org Project gathers the production backlog. Templates, schemas, and the `tbhbdev pm` verbs come from the pm repository.

Every write goes through one content kernel in the tools repository. The kernel renders the text and normalizes it into a block model.

Next, the kernel validates the model against a schema for the type, then scans the text for secrets before any write. A refusal gives the section, the rule, the limit, the actual value, and a hint.

For the operator, the change is one backlog with checked text. Templates stay drafts until the operator says go.

## Proposal

Building the design yields these components, each with one role as owner.

- **The content kernel** (`tbhbdev.content`, in the tools repository, owner tools) creates issues, comments, closes, sub-issue links, and Project entries. It holds the write token, so no plugin can skip the checks.
- **The schemas** (`schemas/` in the pm repository, owner pm) use the Configure Unify Execute (CUE) language. One file per variant, plus a shared core, a taxonomy, and an issue base, define what valid text is.
- **The templates** (`templates/` in the pm repository, owner pm) give starting text for one variant each. Constraints belong to the schemas.
- **The verbs** (`tbhbdev pm`, owner pm) cover create, comment, close, link, and add to Project, on top of the kernel.
- **The org Projects** `work` and `work-calibration` (owner pm) each use the [four views in B. Routing and taxonomy](B-routing-and-taxonomy.md#project-views). The calibration repository uses `work-calibration`.
- **The central work repository** (public, owner pm) and its private twin both hold Initiatives, Epics, and Stories. Bugs and tasks stay with the repositories that contain their code.

Project management and service management use the [shared issue refinement design](../issue-refinement/README.md). See [tracker import](E-import.md) for staged items.

## How it works

### Writing an issue

1. A caller picks a type and variant, then supplies metadata and Markdown that begins with a level-one title.
2. Next, the kernel normalizes the Markdown into the GitHub-flavored block model of sections, blocks, inline counts, and metrics.
3. `cue vet` then checks the model against the schema for that variant and reports every finding in one pass.
4. After that, the kernel runs the secret scan, the quote check, and the prose check on the exact rendered text. A finding in any of them refuses the write. The kernel also runs Vale before every issue body and comment write, with errors downgraded to warnings for short text, as the [shared Vale pass](../issue-refinement/A-refinement-procedure.md#vale-pass) describes.
5. The kernel creates the issue with its type and labels through `create_issue`. Immediately after creation, it uses the draft's `fields` map to write org issue fields on the issue. If a write to a required field fails, the kernel refuses creation. Next, the kernel adds the issue to the repository's Project. Issues in the import calibration repository go only to `work-calibration`.

Each issue template starts with the title line. Use that line as the issue title. Leave it out of the body. Keep copies of metadata out of the body as well.

### Writing guides

The schema is the home for rules about text shape. The few CUE can't express stay in the kernel's named list, and they report in the same refusal format.

A generator builds each guide from the schema. `tbhbdev pm guide <type>` prints the sections in order and marks which ones need text. It also lists caps, metadata rules, and a hint for each section, then shows a rendered example.

Reviewers read the same guide. A refusal uses the guide's wording, which lets a drafter fix the text in one pass.

### Taxonomy

The tools role defines the issue types, field values, type-by-field matrix, and label families in `designs/issue-taxonomy/`. See the [proposed Research variant](A-content-model.md#research).

### Routing

An issue goes in the repository responsible for the work, or in its private twin when the text needs private detail. The private-detail scan picks the twin before any write. Initiatives and Epics go in the public central work repository. Use its private twin when the text needs private detail or the item concerns internal security.

Only people with access to a private repository see its items on the board. See the [routing order](B-routing-and-taxonomy.md#routing-order-for-create).

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| Schema limits are too tight for real issues | Honest writes meet a refusal | Hints in every refusal, limits tuned from first real use, and an issue for each loosening | Unverified until first use |
| The block model counts differ from what a person expects | A refusal looks arbitrary | [A. Content model](A-content-model.md) lists what each metric counts, tested on real output | Proven on one normalizer run |
| The CUE binary is missing where the kernel runs | Checks silently skip | The kernel fails closed when the binary is absent | New |
| A sync job log shows private issue text | Private information leaks | Logs show counts only, and the scan runs before every write | New |
| Kernel checks outside CUE drift from the schemas | A rule gets a second home | A lists each one, and it moves into CUE when the language allows | Existing |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | Kernel write path in tools: create, comment, close, link, and add to Project, each through the scan | The tools tests show a planted secret refuses each write | pm |
| 2 | Schemas and the normalizer, then `cue vet` wiring and the refusal formatter | A mutation test set fails each rule with a named finding | pm |
| 3 | The `guide` generator | `tbhbdev pm guide feature` prints sections, caps, hints, and an example | pm |
| 4 | Templates with schemas, in draft until the operator says go | Each template renders text that passes its own variant | pm |
| 5 | The administrator creates the Source and Status org issue fields. The administrator sets up `work` and `work-calibration`. | Each board has the four views in [B. Routing and taxonomy](B-routing-and-taxonomy.md#project-views). Each view matches the table's layout, filter, grouping, sort order, and visible fields. | pm, administrator |
| 6 | First real items under an Epic in the central work repository or its private twin | The items exist and pass the kernel checks | pm |
| 7 | Sync job | Status follows GitHub state within one run | pm |
| 8 | [Shared refinement](../issue-refinement/README.md) and [batch review](F-refinement-runs.md#review-checklist) | The sync job shows Ready on a test item only with its own current record and, for a batch, a passing review | pm, `sm` |
| 9 | [Tracker import](E-import.md) | Each staged item has a recorded outcome. Each item approved for creation has one issue in its destination type. The import Epic's closing comment gives the outcome counts. | imports |
| 10 | [Remove the retired schema](A-content-model.md#the-files) | The tracker import has finished and the pm role has removed the retired schema. | pm |

## Appendices

- [Decisions](decisions.md) lists one numbered question for the operator, the decided record, and one ask of the tools role.
- [A. Content model](A-content-model.md) says what the block model counts and why the schemas have this structure.
- [B. Routing and taxonomy](B-routing-and-taxonomy.md) covers routing and the Project views.
- [C. Refinement](C-refinement.md) links to shared refinement and the pm batch and import additions.
- [D. Issue writing](D-issue-writing.md) shows how pm applies the shared writing rules and drafts Research issues.
- [E. Tracker import](E-import.md) says how the imports role refines and vets staged files before creating issues.
- [F. Refinement runs](F-refinement-runs.md) gives the run brief, the batch review checklist, and the failure modes of refinement.
- A private appendix covers the private specifics for this design.
