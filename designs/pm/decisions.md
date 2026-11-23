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

### D8. Should refinement send questions of operator intent as one approval Request per batch?

- **Recommendation:** yes. Each batch opens at most one Request that lists each issue with its question, and one comment answers them all.
- **Why:** an import of about 700 issues could otherwise send the operator dozens of separate questions, and the operator already answers Requests in one known place.
- **If yes:** refinement links each Needs info item to the batch Request, and an `sm` run copies each answer onto its issue.
- **If no:** each question goes to the operator as a comment on its own issue, which spreads the questions across many notifications.
- **Reversible:** yes, by changing the refinement brief.

### D9. Do incidents skip refinement?

- **Recommendation:** yes. Response to an incident starts at once, and the problem record and each fix that follow it go through refinement.
- **Why:** waiting for a refinement run would delay the response, while the later fix work gains from the gate like any other item.
- **If yes:** an incident record never waits on a refinement record. Refinement begins with the problem record.
- **If no:** each incident waits for a refinement run first, which adds the length of one run to every response.
- **Reversible:** yes, by changing the exempt list in the refine verb.

## Decided

The operator's records behind D1 to D6 are private notes. Each entry below cites its record by title, with the date, the rule, and the undo path, so a reviewer can judge it without the note. The design relies on D1 to D4 and D6 as requirements. D5 only places the design.

### D7. How does an item become Ready while only the sync job writes Status?

- **Decided:** 2026-11-22, by the pm role.
- **Rule:** the sync job keeps sole write access to every Status value. The refine verb writes a refinement record and a refinement label, and the sync job shows Ready only when a current record says ready and its batch passed review, as [C. Refinement](C-refinement.md) describes.
- **Undo:** let the refine verb write the Ready value itself, which needs a change to D3. Two-way door.
- **Record:** the pull request that adds C-refinement.md to this design.

### D6. Must issues pass refinement before work starts?

- **Decided:** 2026-11-22, by the operator through coordination.
- **Rule:** every issue and every service request passes refinement against a Definition of Ready before it's ready to work, and the `sm` role applies it to service requests. Imported issues enter as not yet ready and pass refinement before work.
- **Undo:** remove the refinement condition from the sync job and the workflow. Two-way door.
- **Record:** the operator's refinement ruling, relayed by coordination.

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
