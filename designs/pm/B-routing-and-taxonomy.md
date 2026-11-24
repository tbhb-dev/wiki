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

Before creating an issue, the kernel scans the exact rendered text. The first matching step wins.

1. If the scan finds a secret, refuse the write and end routing.
2. This step applies when the scan finds private detail, such as private repository names, hosts, or internal paths. An Initiative, Epic, or Feature goes in the central work repository's private twin, including when it concerns an app. A Task, Defect, Chore, Spike, or Research issue about an app goes in the operations repository because the app has no private twin. For other items, use the private twin of the repository that would otherwise hold the issue.
3. An item about [internal security](../issue-refinement/A-refinement-procedure.md#internal-security-topics) goes in a private repository. An Initiative, Epic, or Feature goes in the central work repository's private twin, including when it concerns an app. A Task, Defect, Chore, Spike, or Research issue about an app goes in the operations repository under the app rule in step 2. For other items, use the responsible repository's private twin if it has one. Otherwise, use the central work repository's private twin.
4. A Task, Defect, Chore, Spike, or Research issue with a responsible repository goes in that repository. For an app, write the issue without private detail in the app's own repository.
5. An Initiative, Epic, or Feature goes in the central work repository. Use its private twin when the text needs private detail or the item concerns internal security.
6. Refuse a Task, Defect, Chore, Spike, or Research issue without a responsible repository. Ask the caller to name a parent in the central work repository or its private twin.

The kernel creates the issue in the chosen repository and uses that repository's configured Project. Issues from the calibration repository go only to `work-calibration`. A sub-issue link can tie a work item to its parent across repositories in the same organization. The link keeps a private parent hidden from a public child.

A public Initiative, Epic, or Feature may have private children, and a private one may have Initiative, Epic, or Feature children only when those children are private. A public Task, Defect, Chore, Spike, or Research issue may sit under a private Feature only when its own text contains no private detail.

## One home per fact

In `designs/issue-taxonomy/`, the tools role will define the issue types, fields and values, type-by-field matrix, and label families that pm and `sm` share.

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
