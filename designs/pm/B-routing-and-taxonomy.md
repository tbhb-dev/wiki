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
4. An Initiative or Epic goes in the central work repository, or in its private twin when step 2 picked it.
5. An issue about an internal security topic (a certificate authority, credentials, keys, access control, or host exposure) goes in the private twin without a finding. Refinement moves such an issue to the twin when someone filed it publicly.
6. A work item without a responsible repository gets a refusal, and the caller names a parent in the central work repository.

The kernel creates the issue in the chosen repository, then the work item. A sub-issue link ties a work item to its parent and keeps a private parent hidden from a public child.

## One home per fact

The Project reads the org issue fields and keeps no Project-only field for any fact below.

| Fact | Home | Written by |
| --- | --- | --- |
| Type | Native issue type | The create verb |
| Priority | Org issue field | A verb, with a work note |
| Severity | Org issue field, Defects only | The create verb |
| Work type | Org issue field | The create verb |
| Handler | Org issue field (Operator or Agent), Requests only | The IT service management side |
| Status | Planned org issue field | The sync job for the values that follow GitHub state, and the refinement agent for the readiness values in the refinement appendix |
| Source | Planned org issue field | Not designed yet |
| Area and role | Labels in the `family/value` form | The create verb, with a check against the allowed values in the configuration the tools role designs |
| Owning role | The repository, or a `role/<role>` label for an exception | The create verb |
| Parent | Sub-issue link, shown in the Project's Parent issue field | The link verb |

## Scan before every write

Every write goes through the gitleaks scan on the exact rendered text. The scan has two rule classes. Secret rules refuse the write and show the finding. Private-detail rules choose the twin before any write. The kernel doesn't reword text to pass.

## Public logs

Scheduled jobs run in a public repository, and their logs are public. They show counts and status. Titles, bodies, repository names, and issue numbers stay out.
