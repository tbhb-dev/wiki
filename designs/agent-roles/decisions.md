---
type: Design Appendix
title: Agent roles decisions
description: The open and decided questions of the agent roles design.
tags: [agent-roles, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Agent roles decisions

Back to [the design](README.md).

## Open

### D1. May the coordinator add a standard owner role for each new repository without a Request to you?

- **Recommendation:** yes, as a standing authorization limited to the standard owner entry in [appendix C](C-registry.md#todays-entries). Any role with authority beyond that entry still comes to you.
- **Why:** the constitution reserves adding a role, and new repositories arrive often. Today a new repository's owner is named by the coordinator's reversible call or at your ask (appendix A, R14), and the registry would turn each coordinator call into an ask. The constitution asks for a standing authorization when an ask recurs.
- **If yes:** the registry check accepts a new owner entry that matches the standard template, with the coordinator's record. Any other new role or widened line needs your record.
- **What the registry generates for a new owner entry:** a `role:<name>` label for agent Requests, not a Handler field value. The Handler organization issue field limits how many values it allows, so it stays at Operator and Agent, and per-repository owner roles can't become values of it.
- **If no:** every new repository's owner entry waits on an Operator Request before the repository gets reviews or standup slots.
- **Reversible:** yes. You withdraw the authorization, and later owner entries come to you again. Entries already added stay until you remove them.

### D2. May a launched model run make the approver's read and run the approval command, in place of the approving session?

- **Recommendation:** yes, once step 1's probe shows the approver's definition binds a launched run. A workflow starts the run, never the owner. Today the approving session makes the read and runs the approval command itself, and the agent account can't run that command.
- **Why:** you decided a deterministic loop replaces the long-lived coordination session (R24) and accepted a review and merge worker that approves rebuilt replacements without a model read (R25). Tools has that loop's runner in shadow mode, with an approval command setting. The full-diff read that the approving session makes today has no such answer, and moving it changes who holds approval, which is reserved to you.
- **If yes:** on each green notice, the loop launches the approver role. The run applies today's checks and holds, and the approval command runs at that exact head only on a pass, from an account that may run it, which the change that builds this names. The approving session then stops approving.
- **If no:** the approving session keeps making the read, and the loop waits on that session for every approval that needs one.
- **Reversible:** yes. The loop stops launching the role, and the session takes the read again.

## Decided

### D7. How does a Request name the role that handles it?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** the Handler organization issue field keeps two values, Operator and Agent. An agent Request carries one `role:<name>` label naming the role that handles it, and the label set is generated from the registry, one label per role. Every handler uses the same template, lifecycle, and command-line interface.
- **Why:** this follows from the operator's feedback on D1. An organization issue field limits how many values it allows, so per-repository owner roles can't each be a value.
- **Departs from:** R20's form, the Handler field alone with one value per agent role and no role label. The two-value field that the operator requests draft in wiki#76 describes stays, and the role label it lacks is added.
- **Undo:** the operator decides again.
- **Record:** the operator's decisions of 2026-11-22, R16 and R20 in [appendix A](A-evidence.md#operator-and-coordinator-records), with the form set by the operator's feedback on D1 in this design's pull request.

### D6. Who approves Normal Changes?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** as the constitution says. The owning role decides and records reversible changes, and the coordinator approves a change that crosses roles. Only the reserved matters reach the operator as a Request. An approval names its authority.
- **Undo:** the operator decides again.
- **Record:** the operator's decision of 2026-11-22, R17 in appendix A.

### D5. Does CODEOWNERS name roles?

- **Decided:** 2026-11-11, by the operator, with the file's shape set by the coordinator.
- **Rule:** every repository keeps one owner-less `*` line. The registry checks that line as is and generates no role owners. The records repository adds the authors allowed for decision auto-merge, by the operator's decision of 2026-11-22.
- **Undo:** the operator restores owner lines. Naming a role as a code owner with required review would add approval authority, which is reserved to the operator.
- **Record:** R3 and R18 in appendix A.

### D4. Who owns this design?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** the harness configuration owner owns this design and the registry. It is written in the public wiki, reviewed, and read by the operator before merge.
- **Undo:** the operator names another owner.
- **Record:** R21 in appendix A.

### D3. Do designs name sessions?

- **Decided:** 2026-11-12, by the operator.
- **Rule:** designs name repositories and roles and event-triggered workflows, never today's sessions. Routing events to named sessions is interim and labeled so.
- **Undo:** the operator decides again.
- **Record:** R4 in appendix A.
