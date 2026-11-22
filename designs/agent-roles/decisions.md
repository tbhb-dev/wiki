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

### D2. May a launched model run make the approver's read and run the approval command, in place of the approving session?

- **Recommendation:** yes, once step 1's probe shows the approver's definition binds a launched run. A workflow starts the run, never the builder run whose change it approves. Today the approving session makes the read and runs the approval command itself, and the agent account can't run that command.
- **Why:** you decided a deterministic loop replaces the long-lived coordination session (R24) and accepted a review and merge worker that approves rebuilt replacements without a model read (R25). Tools has that loop's runner in shadow mode, with an approval command setting. The full-diff read that the approving session makes today has no such answer, and moving it changes who holds approval, which is reserved to you.
- **If yes:** on each green notice, the loop launches the approver role. The run applies today's checks and holds, and the approval command runs at that exact head only on a pass, from an account that may run it, which the change that builds this names. The approving session then stops approving.
- **If no:** the approving session keeps making the read, and the loop waits on that session for every approval that needs one.
- **Reversible:** yes. The loop stops launching the role, and the session takes the read again.

## Decided

### D1. Who may add a role for a new repository?

- **Decided:** 2026-11-22, by the operator, answering this design's open question.
- **Rule:** no role is assigned to a repository. The registry defines role kinds and their capabilities only, and creating or widening a kind or a capability stays the operator's. An agent looks up the context each task needs, so a new repository needs no new role. Today's per-repository sessions are interim and retire as loops replace them.
- **Routing:** the Handler field stays Operator or Agent. An agent Request's `area:<repo>` label scopes which repository's context a run loads. Its `cap:<name>` labels, from a short fixed vocabulary, suggest a kind: routing matches them to a kind, the run or router may relabel or reassign, and routing never blocks on them. With no match, a workflow launches the fallback builder kind.
- **Undo:** the operator decides again.
- **Record:** the operator's decisions of 2026-11-22, R27 to R30 in [appendix A](A-evidence.md#operator-and-coordinator-records). R30 superseded the assignment half of R27.

### D7. How does a Request reach the role that handles it?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** the Handler organization issue field keeps two values, Operator and Agent. An agent Request carries an `area:<repo>` label for the context a run loads and `cap:<name>` labels that suggest the kind. The registry generates the label set: one `area:` label per tbhb-dev repository and one `cap:` label per capability in the vocabulary. A check refuses only a `cap:` label outside the vocabulary. Every handler uses the same template, lifecycle, and command-line interface.
- **Why:** an organization issue field limits how many values it allows, so per-repository roles can't each be a value. Capabilities stay few, and no label names a handler.
- **Replaces:** R20's form, the Handler field alone with one value per agent role, and the label forms that followed it the same day (D1). The two-value field that the operator requests draft in wiki#76 describes stays.
- **Undo:** the operator decides again.
- **Record:** the operator's decisions of 2026-11-22, R16, R20, and R27 to R30 in [appendix A](A-evidence.md#operator-and-coordinator-records).

### D6. Who approves Normal Changes?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** as the constitution says. The role making a change decides and records it when it's reversible, and the coordinator approves a change that crosses roles. The record names the owning role, which the builder kind holds per Request in this design. Only the reserved matters reach the operator as a Request. An approval names its authority.
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
