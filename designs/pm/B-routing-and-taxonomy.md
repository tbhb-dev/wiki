---
type: Design Appendix
title: Project management routing and taxonomy
description: The routing order for new items, how the scan picks a twin, and the fields and labels for each fact.
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

1. The kernel scans the exact rendered text before it creates anything. A secret finding refuses the write. The kernel doesn't reroute it to a twin.
2. A private-detail finding (a private repository name, a host, or an internal path) picks the private twin.
3. An item about [internal security](C-refinement.md#internal-security-topics) goes in the private twin regardless of the scan result. If the responsible repository has no private twin, the item goes in the central work repository.
4. For other items, a Defect, Chore, Feature, or Spike goes in the responsible repository or the private twin that step 2 picked.
5. An Initiative or Epic goes in the central work repository.
6. A work item without a responsible repository gets a refusal, and the caller names a parent in the central work repository.

The kernel creates the issue in the chosen repository and uses that repository's configured Project. The import calibration repository routes only to `work-calibration`. A sub-issue link ties a work item to its parent and keeps a private parent hidden from a public child.

## One home per fact

| Fact | Home | Written by |
| --- | --- | --- |
| Type | Native issue type | The create verb |
| Priority | Org issue field | The kernel at creation, then the priority verb with a work note |
| Status | Org issue field derived from [GitHub state](C-refinement.md#status-and-the-ready-signal) | The sync job only |
| Severity | Org issue field for Defects with the value `SEV1`, `SEV2`, or `SEV3` | The kernel at creation |
| Handler | Org issue field for Requests with the value `Agent` or `Operator` | The kernel from the Request draft |
| Work type | Org issue field with the value `Planned` or `Unplanned` where the variant uses it | The kernel at creation |
| Source | Single-select org issue field with the value `Import`, `Review`, `Incident`, or `Backport` | The kernel at creation |
| Area | `area/<subsystem>` label | The issue write commands |
| Concern | `concern/<reason>` label for a cross-cutting reason | The issue write commands |
| Owning role | `role/<role kind>` label with a kind from the agent roles registry | The issue write commands |

All `fields` entries are org issue fields. The kernel's `create_issue` operation writes them on the issue and refuses creation if it cannot write a required field. The issue body holds no metadata copy.

Priority sorts by option order, with `Expedite` first, `Intangible` second, and `Standard` third. Expedite means the cost of delay is immediate, as in an outage or a blocked pipeline. Standard is the default. Intangible means delay has no cost today, as with routine maintenance, cosmetic cleanup, or an improvement nothing waits on. Deferred security work is Standard because exposure grows while it waits.

Source records how the issue was created. Every imported issue has Source `Import`. For an imported review finding, Context cites the pull request or finding. Source `Review` applies when a review run files an issue itself. A deferred review finding that a review run files is a Chore or Defect with Source `Review`.

Security work carries `concern/security`. The merged global and repository configuration lists the allowed labels and field values per issue type. The tools issue write commands refuse any label or field value that configuration disallows.

## Project views

The `work` Project displays these five views using the org issue fields. The `work-calibration` Project uses the same fields and views. The boards read the fields from the issues themselves. Status filters use the values in [C. Refinement](C-refinement.md#status-and-the-ready-signal).

| View | Layout | Filter | Group by | Sort | Visible fields |
| --- | --- | --- | --- | --- | --- |
| Backlog | Table | Open items | Status | Priority, then oldest first | Title, Repository, Priority, Work type, Severity, Assignees |
| Board | Board | Every item except Status `Dropped` | Status (columns) | Priority | Repository, Priority, Severity, Assignees |
| Needs attention | Table | Open items with Status `Needs info` or empty Status | Repository | Oldest first | Title, Status, Labels, Repository |
| Imported | Table | Source `Import` | Repository | Priority | Title, Status, Priority, Work type, Severity |
| Requests | Table | Handler is set | Handler | Priority | Title, Handler, Status, Priority |

## Scan before every write

Every write goes through the gitleaks scan on the exact rendered text. The scan has two rule classes. Secret rules refuse the write and show the finding. Private-detail rules choose the twin before any write. The kernel doesn't reword text to pass.

## Public logs

Scheduled jobs run in a public repository, and their logs are public. They show counts and status. Titles, bodies, repository names, and issue numbers stay out.
