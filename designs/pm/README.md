---
type: Design Draft
title: Project management for tbhb-dev
description: How issues, one org Project, and a checked content kernel give the org a backlog that agents and people write the same way.
tags: [pm, issues, projects, content]
status: draft
owner: pm
updated: "2026-11-22"
decisions_open: 0
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Project management for tbhb-dev

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | pm | 2026-11-22 | Nothing. Read the schemas beside [the content model](A-content-model.md). |

## Summary

Work items are GitHub issues in the repository responsible for the work, and one org Project gathers them with Priority and Status fields. The pm repository supplies the templates, the schemas, and the `tbhbdev pm` verbs.

Every write goes through one content kernel in the tools repository. The kernel renders the text and normalizes it into a block model. It validates the model against a schema for the type, then scans the text for secrets before any write. A refusal gives the section, the rule, the limit, the actual value, and a hint.

For the operator, the change is one backlog with checked text. Templates stay drafts until the operator says go.

## Proposal

Building the design yields these components, each with one role as owner.

- **The content kernel** (`tbhbdev.content`, in the tools repository, owner tools) creates issues, comments, closes, sub-issue links, and Project entries. It holds the write token, so no plugin can skip the checks.
- **The schemas** (`schemas/` in the pm repository, owner pm) use the Configure Unify Execute (CUE) language. One file per variant, plus a shared core, a taxonomy, and an issue base, define what valid text is.
- **The templates** (`templates/` in the pm repository, owner pm) give starting text for one variant each. Constraints belong to the schemas.
- **The verbs** (`tbhbdev pm`, owner pm) cover create, comment, close, link, and add to Project, on top of the kernel.
- **The org Project** (owner pm, with the administrator creating the App) has Priority, Status, and Handler fields and a basic backlog view. Only a sync job writes Status, from GitHub state.
- **The central work repository** (private, owner pm) stores Initiatives and Epics. Bugs and tasks stay with the repositories that contain their code.

## How it works

### Writing an issue

1. A caller picks a type and variant, then supplies metadata and Markdown that begins with a level-one title.
2. Next, the kernel normalizes the Markdown into the GitHub-flavored block model of sections, blocks, inline counts, and metrics.
3. `cue vet` then checks the model against the schema for that variant and reports every finding in one pass.
4. After that, the kernel runs the secret scan, the quote check, and the prose check on the exact rendered text. A finding in any of them refuses the write.
5. The kernel creates the issue and sets its type and fields. Then it adds the issue to the Project.

Each issue template starts with the title line. The kernel takes it as the issue title and removes it from the body.

### Writing guides

The schema is the only home for a rule about text shape, and the generator builds each guide from it. `tbhbdev pm guide <type>` prints the sections in order and marks which ones need text. It also lists caps, metadata rules, and a hint for each section, then shows a rendered example. Reviewers read the same guide. A refusal uses the guide's wording, which lets a drafter fix the text in one pass.

### Taxonomy

- **Handler** (Operator or Agent) applies to Requests only.
- **Severity** runs SEV1 to SEV3 and applies to Defects.
- **Priority** is a Project field (`Expedite`, `Standard`, or `Intangible`) and never a label.
- **Status** comes only from the sync job.
- **`area:<repo>`** labels get a live check against the repository list. **`role:<name>`** labels get a check against a list kept in the private repository for IT service management.

### Routing

An issue goes in the repository responsible for the work, or in its private twin when the text needs private detail. The private-detail scan picks the twin before any write. Larger units go in the central work repository, and a Defect or Chore links to its parent as a sub-issue. The org Project gathers every item, and only people with access to a private repository see its items.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| Schema limits are too tight for real issues | Honest writes meet a refusal | Hints in every refusal, limits tuned from first real use, and a debt row for each loosening | Unverified until first use |
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
| 5 | Org Project with Priority, Status, and Handler, plus a backlog view | An issue added by the kernel shows its fields | pm, administrator |
| 6 | First real items under an Epic in the central work repository | The items exist and pass the kernel checks | pm |
| 7 | Sync job and tracker import | Status follows GitHub state within one run | pm |

## Appendices

- [Decisions](decisions.md): 0 open, then the decided record.
- [A. Content model](A-content-model.md): what the block model counts and why the schemas have this structure.
- [B. Routing and taxonomy](B-routing-and-taxonomy.md): routing, labels, and fields.
- A private appendix covers the private specifics for this design.
