---
type: Writing Checklist
title: Runbook writer checklist
description: The rules a runbook author follows before asking for review.
tags: [runbooks, writing, checklist]
status: draft
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:50691a4df0360088
    at: "2026-11-22T05:45:15Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Runbook writer checklist

Follow this list before asking for review of a runbook. Use the skeleton in [TEMPLATE.md](TEMPLATE.md). The reviewer checks the same ground with [REVIEW.md](REVIEW.md).

## Place and shape

- [ ] The runbook is one file at `runbooks/<role>/<procedure>.md`, or in `runbooks/process/` when it spans roles. The filename states the procedure.
- [ ] It follows the skeleton: frontmatter, then Purpose, Trigger, Preconditions, Steps, What each outcome means, Failure handling and undo, and Related.
- [ ] The frontmatter is complete: `type: Runbook`, `title`, `description`, `tags`, `status`, `owner`, and `updated`. A `Last verified:` line follows the title.
- [ ] The file stays well under 30,000 bytes. A longer procedure splits into runbooks that link each other.

## Truth

- [ ] You ran every step against the live host or tool, and the `Last verified:` line carries that date, or `never` when nobody has run it. A step you couldn't run says so beside the step.
- [ ] Each command sits on its own line in a code block, exact enough to copy and run, with each placeholder named.
- [ ] Each step says what the output looks like when it worked.
- [ ] A helper script is a real file beside the runbook, not a snippet that the reader retypes.
- [ ] A planned tool or check reads as planned, never as something that exists today.

## Roles and privacy

- [ ] The runbook names roles and durable components, never a session or a model run.
- [ ] A public runbook has no secrets, no public IP addresses, no private repository name, no private host name, and no tailnet detail. Private specifics go in a private appendix with the same slug, and the public page says one exists.
- [ ] It quotes no words of the operator verbatim.

## Failure and undo

- [ ] Each step that can fail says how to tell and what to do next.
- [ ] The runbook says how to undo the change, or says it's a one-way door and why.

## Style

- [ ] Each Markdown paragraph is on one line and runs 3 sentences or fewer. Headings use sentence case.
- [ ] Each file spells out an acronym at its first use there, and lists use the Oxford comma.
- [ ] `bazelisk test //...` passes.

## Review and the PR

- [ ] Two reviews from models that didn't write the runbook clear it of blocking items at the exact head they read.
- [ ] The PR body fills every section of the PR template.
