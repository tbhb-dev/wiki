---
type: Runbook
title: Write and review a design
description: How a design goes from intake to tracked work, run by the design owner with an author, two reviewers, and the operator.
owner: design owner
updated: "2026-11-22"
tags: [runbooks, designs, process]
status: draft
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:50691a4df0360088
    at: "2026-11-22T06:44:57Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Write and review a design

Last verified: never.

## Purpose

A design records a choice before the work starts, so reviewers and the operator judge the plan and not the finished change. This runbook covers the path from intake to tracked work. The templates it uses are in [designs/](../../designs/README.md).

## Trigger

Run it when someone asks for a design, or when a change is large enough that a plan has to land first. Don't run it for a small change that a pull request description can explain.

## Preconditions

- A design owner is named. The owner writes the brief, picks the models, and sends the design to the operator.
- The author's session has the repository checked out in its own worktree and has the gates installed.
- The models roster in [models/README.md](../../models/README.md) is current, because it decides who writes and who reviews.

## Steps

1. The design owner takes the request and writes a brief: the question, the audience, the decisions the operator will make, and the files to read.

2. The design owner launches the author on a planning-and-design model from the roster, such as Opus 5.5 or Fable 5.2, at medium effort. The example uses Opus 5.5. While the roster's Antigravity trial runs, it sends new design work to Antigravity first, so check the roster before launching.

   ```sh
   agent exec --harness claude --model claude-opus-5-5 --effort medium
   ```

   The author copies [designs/TEMPLATE.md](../../designs/TEMPLATE.md) and follows [designs/WRITING.md](../../designs/WRITING.md). Bulk reading, such as a survey of many files, goes to a read-only run that the roster picks, and the author reads its report.

3. The author runs the repository gates and opens a pull request with the design.

4. The design owner sends the head to the reviewer role. Two reviewers read the whole design against [designs/REVIEW.md](../../designs/REVIEW.md). The pair follows the author under the roster's review-pair rule. A design from a Claude or Codex session gets Codex `gpt-6.1-sol` and Antigravity `gemini-4-argon-high`. A design with any Antigravity-written part gets Codex and a Claude review run on Sonnet. A reviewer reports findings as blocking or non-blocking, and the verdict covers only the head it read.

5. The author fixes each blocking finding and pushes a new head, which voids the earlier verdicts. The reviewers read the new head.

6. When both reviews carry no blocking finding, the design owner moves the open decisions to the operator, each with a recommendation. The operator answers each one.

7. The author moves each answered item to `## Decided` in the existing design pull request and updates `decisions_open` in the README and `Open decisions: N` in the PR body to match `decisions.md`, and pushes a new head. The reviewers read that head again.

8. The merger merges the pull request on a green `conclusion` at the exact head, after both reviewers clear that head.

9. The design owner turns the plan steps into tracked work in the tracker for the area, each with an owner and an evidence column.

## What each outcome means

| Outcome | Meaning | Next |
| --- | --- | --- |
| A reviewer reports a blocking finding | The design has a defect the operator shouldn't see. | Step 5. |
| Both reviews carry only TECH-DEBT | The design is sound. | Step 6, and file the debt. |
| The operator declines a recommendation | The decision goes the other way. | Step 7 with the operator's answer, then the new head's reviews. |
| The reviewers disagree | One found what the other missed. | The heavier review applies. Step 5. |

## Failure handling and undo

A run that stalls or fails goes to the same role on a fallback model from the roster, and the owner records why. A merged design is undone by a new design that supersedes it, since a decided entry stays as the record of its date.

## Related

- [designs/README.md](../../designs/README.md) lists the template set.
- [runbooks/README.md](../README.md) describes the runbook layout.
- A private appendix holds the host-specific details for this procedure.
