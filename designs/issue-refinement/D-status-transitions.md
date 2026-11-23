---
type: Design Appendix
title: Status transitions
description: When the sync job writes Status and what makes a refinement record current.
tags: [issues, refinement, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Status transitions

Back to [the design](README.md).

The tools role defines Status values and meanings in `designs/issue-taxonomy/`. The rules here tell the sync job when to set an issue's Status. The service-management workflow's Request states appear in the [Request readiness check](B-definition-of-ready.md#requests).

## Transition order

Only the sync job writes the org issue field. It evaluates the issue in the order below and applies the first matching rule.

1. When GitHub closes the issue as completed, set `Done`.
2. For a close as not planned or as a duplicate, set `Dropped`.
3. Clear Status while the issue is open with a current `drop` record. The refine verb retries the close on resumption.
4. Until the kernel checks an issue created outside its write path, set `Unverified`. Use the same value while a tracker item awaits a verified staging or refinement record.
5. On an open issue with a current `needs-info` record, set `Needs info`.
6. Set `In progress` when an open issue has a current `ready` record, any applicable batch review passed, and the issue has an assignee or a linked open pull request.
7. Set `Ready` when an open issue has a current `ready` record, any applicable batch review passed, and the issue has neither an assignee nor a linked open pull request.
8. Set `Needs refinement` on other verified open issues. This includes issues with a `split` record and issues with a `ready` record whose batch review has yet to pass.

## Current records

Use the latest record that the refine verb posts from the kernel's account. It remains current while the body hash matches and the issue has no later return to the queue. For a `ready` record, live fields, labels, and relations must also pass [check 5](B-definition-of-ready.md#readiness-checks).

For a tracker import, the sync job can use the [staging result](../pm/E-import.md#review-and-creation) until ordinary refinement writes a newer record or returns the item to the queue. Accept the staging result only after a passing batch review and a match on both the issue link and body hash.

Recheck item 5 when a live field, label, parent, sub-issue, or dependency changes. Losing a required parent makes a `ready` record stale. A failure of check 5 leaves a current `needs-info` record current. On a checkbox edit, the job rereads the body and checks its hash. It also rereads the applicable fields, labels, and relations.

A reply in a comment leaves the current record valid. See the [question comment procedure](A-refinement-procedure.md#outcome-writes) for the edits and returns that trigger reevaluation.

After a split, the parent needs its own current `ready` record before it can show `Ready` or `In progress`. Recheck the parent after its children together cover its outcome.

## Sync triggers and setup

Issue, issue comment, and pull request events trigger the sync job. GitHub skips new runs for events from a workflow token. A scheduled sync catches those changes.

The administrator turns off built-in Status workflows on both boards. On the next run, the job restores the derived value if someone edits the field by hand. Issue fields and refinement records remain readable when a Project is unavailable.

A worker who finds a Ready item unclear returns it with `tbhbdev pm refine --return` and names the gap in a comment. The return makes the record stale and the job sets Status to `Needs refinement`.
