---
type: Design Appendix
title: Project management decisions
description: The open and decided questions of the project management design.
tags: [pm, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Project management decisions

Back to [the design](README.md).

## Open

None.

## Decided

### D5. Where do the pm and IT service management designs go?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** both designs go in the public wiki under `designs/`, as wiki pull requests under the wiki's protocol. Text describes private repositories by role.
- **Undo:** move the folder. Two-way door.
- **Record:** the operator's decision entry on the public location of the two designs.

### D4. How do the writing rules stay in one place?

- **Decided:** 2026-11-22, by the operator through coordination.
- **Rule:** the schema variant is the only source. Hints sit as data in the schema, a generator prints the guide, and a refusal gives the section, the rule, the limit, the actual value, and the hint.
- **Undo:** replace the generator with hand-written guides. Two-way door.
- **Record:** the operator's writing-guide plan, relayed by coordination.

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
