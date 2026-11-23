---
type: Design Draft
title: Project management for tbhb-dev
description: How issues, one org Project, and a checked content kernel give the org a backlog that agents and people write the same way.
tags: [pm, issues, projects, content]
status: draft
owner: pm
updated: "2026-11-23"
decisions_open: 0
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
| Draft | pm | 2026-11-23 | Nothing. Read the schemas beside [the content model](A-content-model.md). |

## Summary

Work items are GitHub issues in the repository responsible for the work, and one org Project gathers them and reads their org issue fields, such as Priority and Status. The pm repository supplies the templates, the schemas, and the `tbhbdev pm` verbs.

Every write goes through one content kernel in the tools repository. The kernel renders the text and normalizes it into a block model. It validates the model against a schema for the type, then scans the text for secrets before any write. A refusal gives the section, the rule, the limit, the actual value, and a hint.

For the operator, the change is one backlog with checked text. Templates stay drafts until the operator says go.

## Proposal

Building the design yields these components, each with one role as owner.

- **The content kernel** (`tbhbdev.content`, in the tools repository, owner tools) creates issues, comments, closes, sub-issue links, and Project entries. It holds the write token, so no plugin can skip the checks.
- **The schemas** (`schemas/` in the pm repository, owner pm) use the Configure Unify Execute (CUE) language. One file per variant, plus a shared core, a taxonomy, and an issue base, define what valid text is.
- **The templates** (`templates/` in the pm repository, owner pm) give starting text for one variant each. Constraints belong to the schemas.
- **The verbs** (`tbhbdev pm`, owner pm) cover create, comment, close, link, and add to Project, on top of the kernel.
- **The org Project** (owner pm, with the administrator creating the App) reads the org issue fields and has a basic backlog view. A view over Handler serves as the operator's queue.
- **The central work repository** (public, owner pm) stores Initiatives and Epics, and its private twin holds their private detail. Bugs and tasks stay with the repositories that contain their code.

## How it works

### Writing an issue

1. A caller picks a type and variant, then supplies metadata and Markdown that begins with a level-one title.
2. Next, the kernel normalizes the Markdown into the GitHub-flavored block model of sections, blocks, inline counts, and metrics.
3. `cue vet` then checks the model against the schema for that variant and reports every finding in one pass.
4. After that, the kernel runs the secret scan, the quote check, and the prose check on the exact rendered text. A finding in any of them refuses the write.
5. The kernel creates the issue and sets its type and fields. Then it adds the issue to the Project.

Each issue template starts with the title line. The kernel takes it as the issue title and removes it from the body.

### Writing guides

The schema is the home for rules about text shape. The few CUE can't express stay in the kernel's named list, and they report in the same refusal format. The generator builds each guide from the schema. `tbhbdev pm guide <type>` prints the sections in order and marks which ones need text. It also lists caps, metadata rules, and a hint for each section, then shows a rendered example. Reviewers read the same guide. A refusal uses the guide's wording, which lets a drafter fix the text in one pass.

### Taxonomy

- **Priority**, **Severity**, **Work type**, and **Handler** are org issue fields, and the Project reads them. **Source** and **Status** are planned as org issue fields too.
- **Priority** takes `Expedite`, `Standard`, or `Intangible`, and it's never a label.
- **Severity** runs SEV1 to SEV3 and applies to Defects.
- **Handler** takes Operator or Agent and applies to Requests only.
- **Status** has one writer per value. The sync job writes the values that follow GitHub state. The refinement agent writes the readiness values on open issues, such as Needs info and Ready, and the refinement appendix lists them.
- **Labels** take the form `family/value`, such as `area/<repo>` and `role/<role>`. The tools role designs the configuration that lists the allowed values, and the kernel checks each label against it.
- **Owning role** has no field. The repository implies it, and a `role/<role>` label marks an exception.

### Routing

A public issue goes in the repository responsible for the work, and its private detail goes in that repository's private twin. The private-detail scan picks the twin before any write. Larger units go in the central work repository, and their private detail goes in its private twin.

An issue about an internal security topic, such as a certificate authority, credentials, keys, access control, or host exposure, always goes in a private twin. Refinement moves such an issue to the twin when someone filed it publicly.

The service management role (sm) owns the Problem issue type. A Defect's parent is a Problem, an Epic, or nothing. A Problem usually parents the Defects and Chores that fix it, and an Incident links to its Problem.

A child links to its parent as a sub-issue, and the Project shows the parent in its Parent issue field. The org Project gathers every item, and only people with access to a private repository see its items.

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
| 5 | Org Project that reads the org issue fields, plus a backlog view and a view over Handler | An issue added by the kernel shows its fields | pm, administrator |
| 6 | First real items under an Epic in the central work repository | The items exist and pass the kernel checks | pm |
| 7 | Sync job and tracker import | The Status values that follow GitHub state change within one run | pm |

## Appendices

- [Decisions](decisions.md): 0 open, then the decided record.
- [A. Content model](A-content-model.md): what the block model counts and why the schemas have this structure.
- [B. Routing and taxonomy](B-routing-and-taxonomy.md): routing, labels, and fields.
- A private appendix covers the private specifics for this design.
