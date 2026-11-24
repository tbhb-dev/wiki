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

This decision covers ordinary refinement. The tracker import records gaps in staging. A created issue that still needs an answer has Status `Needs info`. See [decision D11](#d11-which-issue-types-does-the-tracker-import-create).

- **Recommendation.** Open at most one approval Request per batch. The Request lists each issue with its question. One comment answers them all.
- **Why.** A batch can contain more than one question of intent. The operator already answers approval Requests in one place.
- **If yes.** Refinement links each item with a question for the operator to the batch Request. An `sm` run copies each answer onto its issue with the question comment link, question number, and source reply link. The `sm` run returns the issue to the queue once every question has an answer. The sync job shows Needs refinement. The next refinement run selects the issue and reads the answers there.
- **If no.** The operator answers the question comment on its issue, as [A. Refinement procedure](../issue-refinement/A-refinement-procedure.md#outcome-writes) describes.
- **Reversible.** Change the refinement brief to switch between the two options.

## Decided

The operator's records behind D1 to D6 are private notes. Each entry below cites its record by title, with the date, the rule, and the undo path, so a reviewer can judge it without the note.

The design relies on D1 to D4 and D6 as requirements. D5 only places the design.

### D9. Do Incidents and Changes skip refinement?

- **Decided.** The operator ruled on 2026-11-23.
- **Rule.** The [service readiness checks](../issue-refinement/B-definition-of-ready.md#records-opened-complete) give the exemption for Incidents and Changes.
- **Undo.** Change the record types that skip the gate.
- **Record.** Issue refinement becomes its own shared design.

### D11. Which issue types does the tracker import create?

- **Decided.** tbhb, 2026-11-23, approved direct creation in the destination issue type.
- **Amended.** On 2026-11-23, the operator ruled that Source records where the work came from. The kernel omits an import marker from the issue fields. Each board uses the [four remaining views](B-routing-and-taxonomy.md#project-views). See the tools design in `designs/issue-taxonomy/` for permitted Source values. The pm proposal there awaits operator confirmation.
- **Owner changes.** The imported schema variant in pm and the taxonomy in tools still use the retired import marker. Their owners update those schemas separately.
- **Rule.** The imports role maps each staged file in one run. A separate run of the imports role refines and vets the file. The kernel creates each approved item once in its destination type. Splits and drops stay in staging. See [E. Tracker import](E-import.md#review-and-creation).
- **Why.** Staging already produces the sections each type requires. Direct creation keeps those sections and avoids about 2,000 follow-up writes.
- **Undo.** Change the mapping and creation path for a future import. Two-way door.
- **Record.** The [tracker import design](E-import.md) gives the approved procedure.

### D7. How does an item become Ready while only the sync job writes Status?

- **Decided:** 2026-11-22, by the pm role.
- **Rule.** The sync job derives Status as [shared Status transitions](../issue-refinement/D-status-transitions.md) describes.
- **Undo.** Let the refine verb set Status. Two-way door.
- **Record.** The original pm refinement design.

### D6. Must issues pass refinement before work starts?

- **Decided:** 2026-11-22, by the operator through coordination.
- **Amended.** On 2026-11-23, the operator moved the gate into the shared design. Requests use workflow validation, Problems have a readiness check, and Incidents and Changes skip refinement.
- **Rule.** The [shared Definition of Ready](../issue-refinement/B-definition-of-ready.md) defines the gate for work items. It also gives the [service checks](../issue-refinement/B-definition-of-ready.md#service-requests-and-problems) and the types that skip refinement. The imports role applies the gate in staging.
- **Undo:** remove the refinement condition from the sync job and the workflow. Two-way door.
- **Record:** the operator's refinement ruling, relayed by coordination.

### D5. Where do the designs for pm and IT service management go?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** both designs go in the public wiki under `designs/`, as wiki pull requests under the wiki's protocol. Text describes private repositories by role.
- **Undo:** move the folder. Two-way door.
- **Record:** the operator's decision on where to publish both designs.

### D4. How do the writing rules stay in one place?

- **Decided:** 2026-11-22, by the operator through coordination.
- **Amended:** 2026-11-22 by coordination (the kernel-checked exceptions), after review found the Configure Unify Execute (CUE) language can't express links-only content.
- **Rule:** the schema variant is the only source for the text-shape rules that CUE can express. Text-shape rules that CUE can't express stay in a short named kernel list (links-only sections, note flooding, a cited Request approved, a retrospective request for comments that exists), and each reports through the same refusal format. Rules that need live data or issue relations also stay in that list. Hints sit as data in the schema, a generator prints the guide, and a refusal gives the section, the rule, the limit, the actual value, and the hint.
- **Undo:** replace the generator with hand-written guides, and move each check on the kernel list into the schema or drop it. Two-way door.
- **Record:** the operator's writing-guide plan, relayed by coordination. Coordination's Decisions record the amendment.

### D3. Which taxonomy applies?

- **Decided:** 2026-11-22, by the operator through coordination.
- **Amended.** On 2026-11-23, the operator assigned the shared issue taxonomy design to the tools role.
- **Rule.** Use the [shared taxonomy reference in B. Routing and taxonomy](B-routing-and-taxonomy.md#one-home-per-fact).
- **Undo:** edit the taxonomy schema. Two-way door.
- **Record.** Coordination relayed the operator's decision on the shared issue taxonomy on 2026-11-23.

### D2. Which language holds the constraints?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** templates create starting text, and the schema language decides validity per variant over a normalized block model. The YAML constraint block no longer exists.
- **Undo:** replace the schemas. Costly but two-way.
- **Record:** the decision entry that content templates and constraints use the schema design from the earlier content-constraints draft.

### D1. Where do issues go?

- **Decided:** 2026-11-22, by the operator.
- **Amended.** On 2026-11-24, the operator ruled that the central work repository and its private twin both hold Initiatives, Epics, and Features.
- **Rule.** Initiatives, Epics, and Features go in the central work repository or its private twin when the text needs private detail or the item concerns internal security. Tasks, Defects, Chores, Spikes, and Research issues go in the responsible repository or the private repository selected by the [routing order](B-routing-and-taxonomy.md#routing-order-for-create). The selected repository determines the org Project. Feature is the unit of user-visible work.
- **Undo:** transfer issues. Two-way door.
- **Record.** The operator's project management issues ruling and the 2026-11-24 “Story means Feature” ruling with its work/private twin clarification.

## Asks of other roles

These questions go to another role, not to the operator, so `decisions_open` doesn't count them. The asked role answers, and the answer moves the item to Decided with its number.

### D10. Can the content check use Vale before each body or comment write and report only warnings for short text?

- **Asked of:** the tools role, which maintains the content kernel and `tbhbdev content check`.
- **Recommendation:** yes. Run the required pass as a plugin of the content check so all kernel callers and import staging share one Vale pass.
- **Why:** today Vale runs only on each repository's own Markdown in continuous integration (CI). Vale doesn't run on issue bodies, comments, or staged import files, and two separate copies in refinement and the import would drift.
- **If yes.** The content check runs Vale with the configuration in [the rules repository](https://raw.githubusercontent.com/tbhb-dev/rules/main/.vale.ini) on each body and comment before the kernel writes it. Staging and issue creation use the same pass, including the short-body threshold and semicolon rule. See the [shared Vale pass](../issue-refinement/A-refinement-procedure.md#vale-pass) and [E. Tracker import](E-import.md).
- **If no.** The kernel calls the same Vale pass directly before every issue body and comment write. Staging calls that pass with the same settings as issue creation.
- **Reversible:** yes. If the tools role removes the plugin, the kernel can call the pass directly.
