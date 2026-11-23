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

None.

## Decided

The operator's records behind D1 to D11 are private notes. Each entry below cites its record by title, with the date, the rule, and the undo path, so a reviewer can judge it without the note. The design relies on D1 to D4 and D6 to D11 as requirements, and D5 only places the design.

On 2026-11-23 the operator ruled that the existing GitHub Projects design stands and needs no separate note. D6 to D11 amend it in place where the operator's decisions of that date differ.

### D11. Where do public and private work go?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** a public issue goes in the repository responsible for the work, and its private detail goes in that repository's private twin. Central work follows the same rule: the central work repository is public, and its private twin holds private detail. An issue about an internal security topic (a certificate authority, credentials, keys, access control, or host exposure) always goes in a private twin, and refinement moves it there when someone filed it publicly.
- **Replaces:** the private central work repository in D1.
- **Undo:** transfer the issues back. Two-way door.
- **Record:** the operator's decision entry on public work and its private twin.

### D10. How do Problems relate to other items?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** the service management role (sm) owns the Problem issue type. A Defect's parent is a Problem, an Epic, or nothing. A Problem usually parents the Defects and Chores that fix it, and an Incident links to its Problem. The Project shows the parent in its Parent issue field.
- **Undo:** remove the Problem parent from the kernel's parent check. Two-way door.
- **Record:** the operator's decision entry on Problems.

### D9. What is Handler, and where is the operator's queue?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** Handler is an org issue field with the values Operator and Agent, and it still applies to Requests only. The operator's queue is a Project view over Handler, not a separate note.
- **Undo:** delete the view. Two-way door.
- **Record:** the operator's decision entry on the Handler field and the operator's queue.

### D8. What form do labels take?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** labels take the form `family/value`, such as `area/<repo>` and `role/<role>`. The tools role designs the configuration that lists the allowed values. An issue carries no owner field, because its repository implies the owning role, and a `role/<role>` label marks an exception.
- **Replaces:** the `area:<repo>` and `role:<name>` label forms and their live lists in D3.
- **Undo:** rename the labels. Two-way door.
- **Record:** the operator's decision entry on the label taxonomy.

### D7. Who writes Status?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** each Status value has one writer. The sync job writes the values that follow GitHub state. The refinement agent writes the readiness values on open issues, such as Needs info and Ready, which the refinement appendix lists.
- **Replaces:** the rule in D3 that only the sync job writes Status.
- **Undo:** give every value back to the sync job. Two-way door.
- **Record:** the operator's decision entry on refinement.

### D6. Where do the work item fields live?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** Priority, Severity, Work type, and Handler are org issue fields, not Project-only fields. Source and Status are planned as org issue fields too. The org Project reads these fields.
- **Undo:** add Project fields and copy the values. Two-way door.
- **Record:** the operator's decision entry on org issue fields.

### D5. Where do the pm and IT service management designs go?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** both designs go in the public wiki under `designs/`, as wiki pull requests under the wiki's protocol. Text describes private repositories by role.
- **Undo:** move the folder. Two-way door.
- **Record:** the operator's decision entry on the public location of the two designs.

### D4. How do the writing rules stay in one place?

- **Decided:** 2026-11-22, by the operator through coordination.
- **Amended:** 2026-11-22 by coord (the kernel-checked exceptions), after review found CUE can't express links-only content.
- **Rule:** the schema variant is the only source for the text-shape rules that CUE can express. Text-shape rules that CUE can't express stay in a short named kernel list (links-only sections, note flooding, a cited Request approved, a retrospective RFC that exists), and each reports through the same refusal format. Rules that need live data or issue relations also stay in that list. Hints sit as data in the schema, a generator prints the guide, and a refusal gives the section, the rule, the limit, the actual value, and the hint.
- **Undo:** replace the generator with hand-written guides, and move each named kernel check into the schema or drop it. Two-way door.
- **Record:** the operator's writing-guide plan, relayed by coordination. The amendment is recorded in coordination's Decisions.

### D3. Which taxonomy applies?

- **Decided:** 2026-11-22, by the operator through coordination.
- **Amended:** 2026-11-23 by D6 to D9, which change the homes of the fields, the Status writers, and the label form.
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
- **Amended:** 2026-11-23 by D11, which makes the central work repository public with a private twin.
- **Rule:** every work item is an issue in the repository responsible for it, and the org Project gathers them. Larger units go in a private central work repository. Story isn't a type, so Feature is the unit of user-visible work.
- **Undo:** transfer issues. Two-way door.
- **Record:** the operator's project-management issues ruling and the rename ruling.
