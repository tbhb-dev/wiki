---
type: Design Draft
title: Project management for tbhb-dev
description: How issues, org issue fields, and a checked content kernel give the org a backlog that agents and people write the same way.
tags: [pm, issues, projects, content]
status: draft
owner: pm
updated: "2026-11-23"
decisions_open: 2
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
| Draft | pm | 2026-11-23 | 2 numbered decisions open in [decisions.md](decisions.md), about 2 minutes |

## Summary

Each work item starts as an issue in the GitHub repository responsible for the work. The `work` org Project gathers the production backlog and displays six org issue fields. Templates, schemas, and the `tbhbdev pm` verbs come from the pm repository.

Every write goes through one content kernel in the tools repository. The kernel renders the text and normalizes it into a block model.

Next, the kernel validates the model against a schema for the type, then scans the text for secrets before any write. A refusal gives the section, the rule, the limit, the actual value, and a hint.

For the operator, the change is one backlog with checked text. Templates stay drafts until the operator says go.

## Proposal

Building the design yields these components, each with one role as owner.

- **The content kernel** (`tbhbdev.content`, in the tools repository, owner tools) creates issues, comments, closes, sub-issue links, and Project entries. It holds the write token, so no plugin can skip the checks.
- **The schemas** (`schemas/` in the pm repository, owner pm) use the Configure Unify Execute (CUE) language. One file per variant, plus a shared core, a taxonomy, and an issue base, define what valid text is.
- **The templates** (`templates/` in the pm repository, owner pm) give starting text for one variant each. Constraints belong to the schemas.
- **The verbs** (`tbhbdev pm`, owner pm) cover create, comment, close, link, and add to Project, on top of the kernel.
- **The org Project** `work` (owner pm) displays the six single-select org issue fields in the [five views in B. Routing and taxonomy](B-routing-and-taxonomy.md#project-views). The `work-calibration` org Project uses the same fields and views for the import calibration repository. Both boards read the fields from the issues themselves.
  - **Priority** accepts `Expedite`, `Intangible`, or `Standard` in that option order.
  - **Status** uses the values in the [Status table in C. Refinement](C-refinement.md#status-and-the-ready-signal), including `Unverified`. Only the sync job writes Status.
  - **Handler** accepts `Agent` or `Operator`. It applies only to Requests.
  - **Source** accepts `Import`, `Review`, `Incident`, or `Backport`. The kernel writes `Import` on every issue the import creates.
  - **Work type** accepts `Planned` or `Unplanned`.
  - **Severity** accepts `SEV1`, `SEV2`, or `SEV3`. It applies only to Defects.
- **The central work repository** (private, owner pm) stores Initiatives and Epics. Bugs and tasks stay with the repositories that contain their code.

Priority, Severity, Work type, and Handler already exist as org issue fields. The administrator creates the Source and Status org issue fields. The administrator sets up `work` and `work-calibration` with the views in [B. Routing and taxonomy](B-routing-and-taxonomy.md#project-views).

Each issue and service request passes the Definition of Ready before work starts. The imports role refines tracker items in staging before the kernel creates them in their destination types. See [C. Refinement](C-refinement.md) and [E. Tracker import](E-import.md).

## How it works

### Writing an issue

1. A caller picks a type and variant, then supplies metadata and Markdown that begins with a level-one title.
2. Next, the kernel normalizes the Markdown into the GitHub-flavored block model of sections, blocks, inline counts, and metrics.
3. `cue vet` then checks the model against the schema for that variant and reports every finding in one pass.
4. After that, the kernel runs the secret scan, the quote check, and the prose check on the exact rendered text. A finding in any of them refuses the write. The kernel also runs Vale before every issue body and comment write, with errors downgraded to warnings for short text, as [C. Refinement](C-refinement.md#vale-pass) describes.
5. The kernel's `create_issue` operation creates the issue with its type and labels. It writes each entry in the draft's `fields` map on the issue itself. It refuses creation if it cannot write a required field. It adds the issue to the Project selected by the repository. Issues in the import calibration repository go only to `work-calibration`.

Each issue template starts with the title line. The kernel takes it as the issue title and removes it from the body. The body holds no metadata copy.

### Writing guides

The schema is the home for rules about text shape. The few CUE can't express stay in the kernel's named list, and they report in the same refusal format.

A generator builds each guide from the schema. `tbhbdev pm guide <type>` prints the sections in order and marks which ones need text. It also lists caps, metadata rules, and a hint for each section, then shows a rendered example.

Reviewers read the same guide. A refusal uses the guide's wording, which lets a drafter fix the text in one pass.

### Taxonomy

- **Handler** (Operator or Agent) applies to Requests only.
- **Severity** runs SEV1 to SEV3 and applies to Defects.
- **Priority** is an org issue field. Expedite means the cost of delay is immediate, as in an outage or a blocked pipeline. Standard is the default. Intangible means delay has no cost today, as with routine maintenance, cosmetic cleanup, or an improvement nothing waits on. Deferred security work is Standard because exposure grows while it waits.
- **Status** comes only from the sync job.
- **Labels** use `area/<subsystem>`, `concern/<reason>`, and `role/<role kind>`. The merged global and repository configuration lists allowed labels and field values per issue type.

### Routing

An issue goes in the repository responsible for the work, or in its private twin when the text needs private detail. The private-detail scan picks the twin before any write. Larger units go in the central work repository.

A Chore links to its parent as a sub-issue, and a Defect links to a Problem or an Epic when it has a parent. An issue about internal security always goes in a private repository. Only people with access to a private repository see its items on the board.

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
| 5 | The administrator creates the Source and Status org issue fields. The administrator sets up `work` and `work-calibration` to display the six issue fields. | Each board has all five views listed in [B. Routing and taxonomy](B-routing-and-taxonomy.md#project-views). Backlog shows Priority and groups items by Status. Requests shows Handler on a Request. Imported shows an issue with Source `Import`. Backlog shows Work type on a Task, Feature, Spike, or Chore and Severity on a Defect. | pm, administrator |
| 6 | First real items under an Epic in the central work repository | The items exist and pass the kernel checks | pm |
| 7 | Sync job | Status follows GitHub state within one run | pm |
| 8 | [Refinement](C-refinement.md) and [batch review](F-refinement-runs.md#review-checklist) | The sync job shows Ready on a test item only with its own current record and, for a batch, a passing review | pm, `sm` |
| 9 | [Tracker import](E-import.md) | Each staged item has a recorded outcome. Each item approved for creation has one issue in its destination type. The import Epic's closing comment gives the outcome counts. | imports |
| 10 | [Remove the retired schema](A-content-model.md#the-files) | The tracker import has finished and the pm role has removed the retired schema. | pm |

## Appendices

- [Decisions](decisions.md) holds two numbered questions for the operator, the decided record, and one ask of the tools role.
- [A. Content model](A-content-model.md) says what the block model counts and why the schemas have this structure.
- [B. Routing and taxonomy](B-routing-and-taxonomy.md) covers routing, labels, and fields.
- [C. Refinement](C-refinement.md) gives the Definition of Ready, the refinement procedure, and the Vale pass.
- [D. Issue writing](D-issue-writing.md) gives the writing rules for every issue and the content checks that enforce them.
- [E. Tracker import](E-import.md) says how the imports role refines and vets staged files before creating issues.
- [F. Refinement runs](F-refinement-runs.md) gives the run brief, the batch review checklist, and the failure modes of refinement.
- A private appendix covers the private specifics for this design.
