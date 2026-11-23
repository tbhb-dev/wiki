---
type: Design Appendix
title: Project management decisions
description: The open and decided questions of the project management design.
tags: [pm, decisions, design]
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

# Project management decisions

Back to [the design](README.md).

## Open

The choice between Epics as design documents and an Epic issue type remains open. References to Epic issues in this draft are provisional.

### D8. Should ordinary refinement send questions of operator intent as one approval Request per batch?

This decision covers ordinary refinement. The tracker import records gaps in staging. At creation, the kernel adds `flag:split`, `flag:drop`, or `flag:needs-info` when the staged outcome calls for owner follow-up. See [decision D11](#d11-which-issue-types-does-the-tracker-import-create).

- **Recommendation.** Open at most one approval Request per batch. The Request lists each issue with its question. One comment answers them all.
- **Why.** A batch can contain several questions of intent. The operator already answers approval Requests in one place.
- **If yes.** Refinement links each item with a question for the operator to the batch Request. An `sm` run copies each answer onto its issue with the question comment link, question number, and source reply link. The `sm` run removes the issue's `needs-info` refinement label once every question has an answer. The sync job shows Needs refinement. The next refinement run selects the issue and reads the answers there.
- **If no.** The operator answers the question comment on its issue, as [C. Refinement](C-refinement.md#outcome-writes) describes.
- **Reversible.** Change the refinement brief to switch between the two options.

### D9. Do incidents skip refinement?

- **Recommendation:** yes. Response to an incident starts at once, and the problem record and each fix that follow it go through refinement.
- **Why:** waiting for a refinement run would delay the response, while the later fix work gains from the gate like any other item.
- **If yes:** an incident record never waits on a refinement record. Refinement begins with the problem record.
- **If no:** the `sm` role applies the common readiness checks and the Incident template and schema before response. This is the rule while D9 is open.
- **Reversible:** yes, by changing the exempt list in the refine verb.

## Decided

The operator's records behind D1 to D6 are private notes. Each entry below cites its record by title, with the date, the rule, and the undo path, so a reviewer can judge it without the note.

The design relies on D1 to D4 and D6 as requirements. D5 only places the design.

### D11. Which issue types does the tracker import create?

- **Decided.** tbhb, 2026-11-23, approved direct creation in the destination issue type.
- **Rule.** The imports role refines and vets each file in staging. The kernel writes each issue once in its destination type with Source `Import`. At creation, the kernel adds `flag:split`, `flag:drop`, or `flag:needs-info` to match the staged outcome.
- **Why.** Staging already produces the sections each type requires. Direct creation keeps those sections and avoids about 2,000 follow-up writes.
- **Undo.** Change the mapping and creation path for a future import. Two-way door.
- **Record.** The [tracker import design](E-import.md) gives the approved procedure.

### D7. How does an item become Ready while only the sync job writes Status?

- **Decided:** 2026-11-22, by the pm role.
- **Rule:** the sync job keeps sole write access to every Status value. Ready and In progress both require the item's own current `ready` record, and any batch it belongs to must pass review. An open pull request or an assignee moves an item to In progress only once it meets that gate. [C. Refinement](C-refinement.md) defines when a record is current.
- **Undo:** let the refine verb write the Ready value itself, which needs a change to D3. Two-way door.
- **Record:** the pull request that adds C-refinement.md to this design.

### D6. Must issues pass refinement before work starts?

- **Decided:** 2026-11-22, by the operator through coordination.
- **Rule.** Every issue and every service request passes the Definition of Ready before work starts. The `sm` role refines Requests, Incidents, Problems, and Changes. It applies the common readiness checks and uses each Incident, Problem, or Change template and schema for the type-specific checks. For tracker imports, the imports role applies the gate in staging. The import sets Source to `Import` at creation. That value meets the [Problem Source requirement](../itsm-records/problems.md#fields).
- **Undo:** remove the refinement condition from the sync job and the workflow. Two-way door.
- **Record:** the operator's refinement ruling, relayed by coordination.

### D5. Where do the pm and IT service management designs go?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** both designs go in the public wiki under `designs/`, as wiki pull requests under the wiki's protocol. Text describes private repositories by role.
- **Undo:** move the folder. Two-way door.
- **Record:** the operator's decision entry on the public location of the two designs.

### D4. How do the writing rules stay in one place?

- **Decided:** 2026-11-22, by the operator through coordination.
- **Amended:** 2026-11-22 by coordination (the kernel-checked exceptions), after review found the Configure Unify Execute (CUE) language can't express links-only content.
- **Rule:** the schema variant is the only source for the text-shape rules that CUE can express. Text-shape rules that CUE can't express stay in a short named kernel list (links-only sections, note flooding, a cited Request approved, a retrospective request for comments that exists), and each reports through the same refusal format. Rules that need live data or issue relations also stay in that list. Hints sit as data in the schema, a generator prints the guide, and a refusal gives the section, the rule, the limit, the actual value, and the hint.
- **Undo:** replace the generator with hand-written guides, and move each check on the kernel list into the schema or drop it. Two-way door.
- **Record:** the operator's writing-guide plan, relayed by coordination. Coordination's Decisions record the amendment.

### D3. Which taxonomy applies?

- **Decided:** 2026-11-22, by the operator through coordination.
- **Rule:** handler applies to Requests only, and Severity runs SEV1 to SEV3. Priority is a field and never a label. Only the sync job writes Status. Area labels get a live check in the kernel, and the IT service management side maintains role labels.
- **Undo:** edit the taxonomy schema. Two-way door.
- **Record:** the coordination message of 2026-11-22 that set the taxonomy.

### D2. Which language holds the constraints?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** templates create starting text, and the schema language decides validity per variant over a normalized block model. The YAML constraint block no longer exists.
- **Undo:** replace the schemas. Costly but two-way.
- **Record:** the decision entry that content templates and constraints use the schema design from the earlier content-constraints draft.

### D1. Where do issues go?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** every work item is an issue in the repository responsible for it, and the org Project gathers them. Larger units go in a private central work repository. Story isn't a type, so Feature is the unit of user-visible work.
- **Undo:** transfer issues. Two-way door.
- **Record:** the operator's project-management issues ruling and the rename ruling.

## Asks of other roles

These questions go to another role, not to the operator, so `decisions_open` doesn't count them. The asked role answers, and the answer moves the item to Decided with its number.

### D10. Can the content check run Vale with the org configuration on issue and comment bodies before each write, with errors as its rules set them and warnings only for a short body?

- **Asked of:** the tools role, which maintains the content kernel and `tbhbdev content check`.
- **Recommendation:** yes. Run the required pass as a plugin of the content check so all kernel callers and import staging share one implementation.
- **Why:** today Vale runs only on each repository's own Markdown in continuous integration (CI). Vale doesn't run on issue bodies, comments, or staged import files, and two separate copies in refinement and the import would drift.
- **If yes:** the content check runs Vale with the configuration in [the rules repository](https://github.com/tbhb-dev/rules/blob/main/.vale.ini) on each body and comment before the kernel writes it. Import staging and issue creation use the same pass, including the short-body threshold and semicolon rule, as [C. Refinement](C-refinement.md#vale-pass) and [E. Tracker import](E-import.md) describe.
- **If no:** the kernel calls the same Vale pass directly before every issue body and comment write. Import staging calls that pass with the same settings as issue creation.
- **Reversible:** yes. The kernel can call the pass directly if the plugin is removed.
