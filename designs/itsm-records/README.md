---
type: Design Draft
title: Service-management records
description: Where tbhb-dev keeps its incidents, problems, requests, changes, and decisions as tracked records, and how the operator answers a Request only they can act on.
tags: [service-management, records, issues, design]
status: draft
owner: Service-management maintainer
updated: "2026-11-22"
decisions_open: 0
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:933656c1cea36ed5
    subagent: sha-256-64:ea6e32cf22cbcb34
    at: "2026-11-22T23:15:39Z"
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:933656c1cea36ed5
    at: "2026-11-22T23:15:39Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Service-management records

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Service-management maintainer | 2026-11-22 | Nothing. The operator answered the five questions in [decisions.md](decisions.md) |

## Summary

tbhb-dev's work state used to sit in an unversioned notes folder. This design moves it into GitHub Issues, Projects, and one private records repository, so every item has an owner, a state, a history, and a check that its body fits.

Each kind of record is an issue or a record file with a fixed shape: incident, problem, request, change, note, transition, and decision. A schema variant states the limits of each shape, and the shared content check refuses a body that breaks them before anything is written.

For the operator, the one change is that a Request only they can act on arrives as an issue with an exact command to run, and they answer it with one of four commands.

## Proposal

A plugin of the shared command-line tool opens, updates, and closes the records. The private records repository holds them. A workflow in that repository moves a Request when the operator answers it. This folder describes the pieces that exist as a design. The parts that are only built are labeled in each appendix.

## How it works

Operator Requests are the first part written up. A Request that only the operator can act on says why, shows the exact command, and waits. The operator runs it from their own terminal, and the result returns as one comment from their own login. [Operator requests](D-operator-requests.md) gives the states, the commands, the workflow side, and the threats.

Problems are the second part. A Problem records a cause behind Incidents or a recurring failure and tracks the Defects and Chores that fix it, then ends Resolved once its fixes merge and its Resolution criteria and recurrence check pass, or Accepted on a recorded decision to live with the failure. [Problems](problems.md) gives the fields, the states, the worked splits from the import, and the migration.

The record kinds, the schema variants other than the planned Problem variant, and the decision records are built in the private repository today. Public-safe appendices describing them will be written here later.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| The content check doesn't yet enforce the schema variants | A body that breaks its shape still passes | The kernel adopts the variants, tracked in the private repository | unverified |
| A command Request runs a script the author wrote | The operator runs code nobody reviewed | One dated script per Request, shown in full before the operator confirms | new |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | The record schema variants and templates | Their pull requests merge with green checks | Service-management maintainer |
| 2 | The operator commands | `request run`, `approve`, `deny`, and `comment` each post their comment in a test run | Service-management maintainer |
| 3 | The remaining design appendices, newly written as public-safe material | Each is merged here | Service-management maintainer |

## Appendices

- [Operator requests](D-operator-requests.md): how a Request only the operator can act on is opened, answered, run, and closed.
- [Problems](problems.md): how a Problem tracks the Defects and Chores that fix its cause until it ends as Resolved or Accepted, and how the problems tracker and the staged import become Problems.
- [Decisions](decisions.md): none open, then the decided record.
