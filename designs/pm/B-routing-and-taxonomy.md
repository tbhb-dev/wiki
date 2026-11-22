---
type: Design Appendix
title: Project management routing and taxonomy
description: The routing order for new items, how the scan picks a twin, and the fields and labels for each fact.
tags: [pm, routing, taxonomy, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Project management routing and taxonomy

Back to [the design](README.md).

## Routing order for create

1. A bug or task goes in the repository responsible for the work, or in its private twin when the text needs private detail.
2. An Initiative or Epic goes in the central work repository.
3. A bug or task without a responsible repository gets a refusal, and the caller names a parent in the central work repository.

The kernel creates the issue in the responsible repository first, then in the twin if the scan requires it, then the work item. A sub-issue link ties a bug or task to its parent and keeps a private parent hidden from a public child.

## One home per fact

| Fact | Home | Written by |
| --- | --- | --- |
| Type | Native issue type | The create verb |
| Priority | Project field | A verb, with a work note |
| Status | Project field | The sync job only |
| Severity | Defect field in the body metadata | The create verb |
| Handler | Project field, Requests only | The IT service management side |
| Area and role | Labels | The create verb, with a check against live lists |

## Scan before every write

Every write goes through the gitleaks scan on the exact rendered text. A finding refuses the write and shows the finding. The kernel doesn't reword text to pass. The twin choice uses the same scan.

## Public logs

Scheduled jobs run in a public repository, and their logs are public. They show counts and status. Titles, bodies, repository names, and issue numbers stay out.
