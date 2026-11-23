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
2. A private-detail finding (a private repository name, a host, or an internal path) picks the private twin. No finding picks the public repository.
3. A Defect, Chore, Feature, or Spike goes in the responsible repository, or in its private twin when step 2 picked it.
4. An Initiative or Epic goes in the central work repository.
5. A work item without a responsible repository gets a refusal, and the caller names a parent in the central work repository.

The kernel creates the issue in the chosen repository, then the work item. A sub-issue link ties a work item to its parent and keeps a private parent hidden from a public child.

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

Every write goes through the gitleaks scan on the exact rendered text. The scan has two rule classes. Secret rules refuse the write and show the finding. Private-detail rules choose the twin before any write. The kernel doesn't reword text to pass.

## Public logs

Scheduled jobs run in a public repository, and their logs are public. They show counts and status. Titles, bodies, repository names, and issue numbers stay out.
