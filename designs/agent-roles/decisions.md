---
type: Design Appendix
title: Agent roles decisions
description: The open and decided questions of the agent roles design.
tags: [agent-roles, decisions, design]
status: draft
parent: README.md
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-opus-5-5
    effort: medium
    surface: cli
    session: sha-256-64:e48956ec8ffa792a
    subagent: sha-256-64:63469a419533009b
    at: "2026-11-22T23:16:43Z"
  - role: assisted
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:e48956ec8ffa792a
    at: "2026-11-22T23:16:43Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Agent roles decisions

Back to [the design](README.md).

## Open

None.

## Decided

### D2. May a launched model run make the approver's read and run the approval command, in place of the approving session?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** yes. A launched run of the approver kind may make the approver's read and run the approval command in place of the approving session. One probe run proves it before it replaces the session. The rule that a maker never approves its own change and the provider rules bind the run unchanged, and a workflow starts the run, never the builder run whose change it approves.
- **Undo:** the loop stops launching the kind, and the session takes the read again.
- **Record:** the operator's decision of 2026-11-23, R33 in [appendix A](A-evidence.md#operator-and-coordinator-records).

### D1. Who may add a role for a new repository?

- **Decided:** 2026-11-23, by the operator, answering this design's open question.
- **Rule:** no role is assigned to a repository. The registry defines role kinds and their capabilities only, and creating or widening a kind or a capability stays the operator's. An agent looks up the context each task needs, so a new repository needs no new role. Today's per-repository sessions are interim and retire as loops replace them.
- **Routing:** the Handler field stays Operator or Agent. An agent Request's `area:<repo>` label scopes which repository's context a run loads. Its `cap:<name>` labels, from a short fixed vocabulary, suggest a kind: routing matches them to a kind, the run or router may relabel or reassign, and routing never blocks on them. With no match, a workflow launches the fallback builder kind.
- **Undo:** the operator decides again.
- **Record:** the operator's decisions R27 and R28 of 2026-11-22 and R29 and R30 of 2026-11-23 in [appendix A](A-evidence.md#operator-and-coordinator-records). R30 superseded the assignment half of R27.

### D8. Where do the roles live?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** documentation of the role kinds and capabilities is public, in this wiki. The mechanical parts that go straight into harnesses live in the private harness configuration repository: the registry data, generated agent definitions and profiles, allow and deny entries, and hooks. The operator decided public docs and private mechanics. Generating the public page from that registry data, with a check that fails on drift, is this design's proposal and target design, not built.
- **Undo:** the operator decides again.
- **Record:** the operator's decision of 2026-11-23, R32 in [appendix A](A-evidence.md#operator-and-coordinator-records). It settles the design's earlier proposal of a private home.

### D7. How does a Request reach the role that handles it?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** the Handler organization issue field keeps two values, Operator and Agent. An agent Request carries an `area:<repo>` label for the context a run loads and `cap:<name>` labels that suggest the kind. The registry generates the label set: one `area:` label per tbhb-dev repository and one `cap:` label per capability in the vocabulary. A check refuses only a `cap:` label outside the vocabulary. Every handler uses the same template, lifecycle, and command-line interface.
- **Why:** an organization issue field limits how many values it allows, so per-repository roles can't each be a value. Capabilities stay few, and no label names a handler.
- **Replaces:** R20's form, the Handler field alone with one value per agent role, and the label forms that followed it the same day (D1). The two-value field that the operator requests draft in wiki#76 describes stays.
- **Undo:** the operator decides again.
- **Record:** the operator's decisions R16, R20, R27, and R28 of 2026-11-22 and R29 and R30 of 2026-11-23 in [appendix A](A-evidence.md#operator-and-coordinator-records).

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
