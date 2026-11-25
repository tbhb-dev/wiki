---
type: Design Draft
title: Shared issue refinement
description: How project management and service management check whether an issue is ready for work.
tags: [issues, refinement, design]
status: draft
owner: pm
updated: "2026-11-23"
decisions_open: 1
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Shared issue refinement

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | pm | 2026-11-23 | One open choice in [decisions](decisions.md#open). |

## Summary

Project management (pm) and service management (`sm`) use one design for issue readiness. Workers can look up the requirements and check how the result applies to their issue.

## Proposal

Start with the Definition of Ready for the evidence required before work. Reviewers use the rubric to judge the writing. Maintenance belongs to the pm role. Service workflows and review belong to the `sm` role.

Find the shared taxonomy in `designs/issue-taxonomy/`, which the tools role maintains. Project management adds [batch review and tracker import](../pm/C-refinement.md).

## How it works

A refiner follows the procedure for work items. Service records use the [checks for their type](B-definition-of-ready.md#service-requests-and-problems). The sync job uses the refinement record and GitHub state to derive Status.

## Risks and unknowns

| Risk or unknown | Effect | What handles it | Evidence |
| --- | --- | --- | --- |
| An edit makes the record stale. | Work could start on an earlier draft. | The [sync job checks whether the record is current](D-status-transitions.md#current-records). | Planned. |
| A service variant expects procedure in the issue. | The variant can require content that the shared writing rules reject. | The [decision record](decisions.md#d4-which-service-rules-conflict-with-the-shared-rules) identifies the conflicts for `sm` review. | Existing variants. |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | Shared design and service review. | The service reviewer checks the service rules against its workflows and variants. | pm and `sm`. |
| 2 | Schema and kernel lint. | The lint flags the failing examples in the rubric. | pm and tools. |
| 3 | Status sync. | A test issue's Status follows its current record and GitHub state. | pm. |

## Appendices

- [Decisions](decisions.md) records one open choice and the rulings behind this design.
- [A. Refinement procedure](A-refinement-procedure.md) gives the steps, records, evidence, and Vale pass.
- [B. Definition of Ready](B-definition-of-ready.md) gives sixteen readiness checks and the writing rules, including the deliberate Out of scope decision.
- [C. Writing rubric](C-writing-rubric.md) gives pass and fail tests for issue writing.
- [D. Status transitions](D-status-transitions.md) explains how the job sets Status.
