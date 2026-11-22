---
type: Design Draft
title: Service-management records
description: Where tbhb-dev keeps its incidents, problems, requests, changes, and decisions as tracked records, and how the operator answers a Request only they can act on.
tags: [service-management, records, issues, design]
status: draft
owner: Service-management maintainer
updated: "2026-11-22"
decisions_open: 0
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Service-management records

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Service-management maintainer | 2026-11-22 | Five questions at the end of the operator-requests appendix, about 5 minutes |

## Summary

tbhb-dev's work state used to sit in an unversioned notes folder. This design moves it into GitHub Issues, Projects, and one private records repository, so every item has an owner, a state, a history, and a check that its body fits.

Each kind of record is an issue or a record file with a fixed shape: incident, problem, request, change, note, transition, and decision. A schema variant states the limits of each shape, and the shared content check refuses a body that breaks them before anything is written.

For the operator, the one change is that a Request only they can act on arrives as an issue with an exact command to run, and they answer it with one of four commands.

## Proposal

A plugin of the shared command-line tool opens, updates, and closes the records. The private records repository holds them. A workflow in that repository moves a Request when the operator answers it. This folder describes the pieces that exist as a design. The parts that are only built are labeled in each appendix.

## How it works

Operator Requests are the first part written up. A Request that only the operator can act on says why, shows the exact command, and waits. The operator runs it from their own terminal, and the result returns as one comment from their own login. [Operator requests](operator-requests.md) gives the states, the commands, the workflow side, and the threats.

The record kinds, their schema variants, and the decision records are built in the private repository today and move into this folder as further appendices.

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
| 3 | The remaining design appendices | Each is merged here and the private copies are removed | Service-management maintainer |

## Appendices

- [Operator requests](operator-requests.md): how a Request only the operator can act on is opened, answered, run, and closed.
