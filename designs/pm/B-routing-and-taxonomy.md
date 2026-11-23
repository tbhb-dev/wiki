---
type: Design Appendix
title: Project management routing and taxonomy
description: The routing order for new items, how the scan picks a twin, and the views on both boards.
tags: [pm, routing, taxonomy, design]
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

# Project management routing and taxonomy

Back to [the design](README.md).

## Routing order for create

1. Before creating an issue, the kernel scans the exact rendered text. Refuse the write if the scan finds a secret. End routing.
2. Route private details to the twin. The scan looks for private repository names, hosts, and internal paths.
3. An item about [internal security](../issue-refinement/A-refinement-procedure.md#internal-security-topics) goes in the private twin regardless of the scan result. If the responsible repository has no private twin, the item goes in the central work repository.
4. For other items, a Defect, Chore, Feature, Spike, or Research issue goes in the responsible repository or the private twin that step 2 picked.
5. An Initiative or Epic goes in the central work repository.
6. Refuse a work item without a responsible repository. Ask the caller to name a parent in the central work repository.

The kernel creates the issue in the chosen repository and uses that repository's configured Project. Issues from the calibration repository go only to `work-calibration`. A sub-issue link ties a work item to its parent and keeps a private parent hidden from a public child.

## One home per fact

In `designs/issue-taxonomy/`, the tools role defines the issue types, fields and values, type-by-field matrix, and label families that pm and `sm` share.

Use the priority verb with a work note to change an issue's Priority.

## Project views

The `work` and `work-calibration` org Projects each have four views. Both boards display the org issue fields without storing copies. See the [shared Status transitions](../issue-refinement/D-status-transitions.md).

| View | Layout | Filter | Group by | Sort | Visible fields |
| --- | --- | --- | --- | --- | --- |
| Backlog | Table | Open items | Status | Priority, then oldest first | Title, Repository, Priority, Work type, Severity, Assignees |
| Board | Board | Every item except Status `Dropped` | Status (columns) | Priority | Repository, Priority, Severity, Assignees |
| Needs attention | Table | Open items with Status `Needs info` or empty Status | Repository | Oldest first | Title, Status, Labels, Repository |
| Requests | Table | Handler has a value | Handler | Priority | Title, Handler, Status, Priority |

## Scan before every write

Every write goes through the gitleaks scan on the exact rendered text. The scan has two rule classes. Secret rules refuse the write and show the finding. Private-detail rules choose the twin before any write. The kernel doesn't reword text to pass.

## Public logs

Scheduled jobs run in a public repository, and their logs are public. They show counts and status. Titles, bodies, repository names, and issue numbers stay out.
