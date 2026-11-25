---
type: Design Appendix
title: Shared issue refinement decisions
description: The rulings and open scope question behind shared issue refinement.
tags: [issues, refinement, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Shared issue refinement decisions

Back to [the design](README.md).

## Open

### D5. Where do open scope questions go?

- **Recommendation.** Add a dedicated Open scope questions section to both variants.
- **Why.** Out of scope records settled exclusions. Both variants still lack a section for unresolved scope questions, and the [hedge check](B-definition-of-ready.md#unresolved-hedges) needs a place for them.
- **If yes.** The later pm schema change adds the section.
- **If no.** Resolve scope questions before either type becomes ready.
- **Reversible.** A later schema change can remove the section after the refiner resolves or moves any remaining questions.

## Decided

### D6. Must refinement decide Out of scope?

- **Decided.** The operator ruled on 2026-11-25.
- **Rule.** Out of scope is required in every issue type. The last of the sixteen [readiness checks](B-definition-of-ready.md#readiness-checks) requires a deliberate decision before Ready. List deliberate exclusions, or use one `None.` item after considering what to exclude.
- **Undo.** Change the readiness check and its rubric test together with the section requirement.
- **Record.** The operator's Decisions entry on 2026-11-25 makes considering Out of scope part of refinement. A separate pm change updates the schemas and templates.

### D4. Which service rules conflict with the shared rules?

- **Decided.** On 2026-11-23, the operator directed pm to record the service rules with their conflicts for `sm` review.
- **Rule.** See the [Definition of Ready](B-definition-of-ready.md#service-requests-and-problems) for the service checks. Existing workflow rules come from the service design and schemas.
- **Undo.** Return the service check summary to the service design.
- **Record.** Issue refinement becomes its own shared design.

Earlier pm drafts proposed a refinement run and a ready record for Requests. For Requests, workflow validation serves as refinement. Service labels use lower-case states. For the org field, the sync job writes Status in sentence case. See the org field values in `designs/issue-taxonomy/`. The service text uses `area:<repo>`, capability labels, and `role:<name>` labels. The shared rulings use `area/` and `role/` with role kinds and omit capability labels.

Command Requests require checks, undo, dry-run steps, and a command. Those sections conflict with the no-procedure rule. In the Problem design, Workaround and Recurrence check can also contain procedures. Migration examples for Problems still use the retired import marker. For Problems, the service design calls Priority a Project field, while pm treats it as an org issue field. Those records remain in the service design for `sm` review.

Different schemas define the base, command, and change body shapes. An author edit reopens an active Request for a check. In later states, the [service workflow](../itsm-records/D-operator-requests.md#the-workflow-side) leaves running and terminal states alone. For commands, the schema requires an area label even though the service design says Operator Requests omit it. Command, base, and change schemas also permit Work type. See the tools taxonomy for the field matrix.

### D3. Where does procedure go?

- **Decided.** The operator ruled on 2026-11-23.
- **Rule.** Use the [no-procedure rule](B-definition-of-ready.md#no-procedure) when writing an issue.
- **Undo.** Change the rule and the corresponding rubric test.
- **Record.** `Issues carry no procedure`.

### D2. Where do the key refinement artifacts live?

- **Decided.** The operator ruled on 2026-11-23.
- **Rule.** The Definition of Ready, rubric, and Status transitions each have their own lettered appendix.
- **Undo.** Move the artifacts back into the procedure.
- **Record.** Refinement's key artifacts are their own lettered appendices.

### D1. Who shares the refinement design?

- **Decided.** The operator ruled on 2026-11-23.
- **Rule.** Project management and service management use the shared design. The `sm` role reviews it.
- **Undo.** Move the rules back to the two designs.
- **Record.** Issue refinement becomes its own shared design.
