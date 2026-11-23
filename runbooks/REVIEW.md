---
type: Review Checklist
title: Runbook review checklist
description: What a reviewer checks in a runbook pull request.
tags: [runbooks, review, checklist]
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

# Runbook review checklist

The reviewer didn't write or edit the runbook, and reads the whole file and any helper script beside it. Check each item against the live host or tool and primary sources, not against the author's summary. The rules are in [README.md](README.md), [TEMPLATE.md](TEMPLATE.md), and [WRITING.md](WRITING.md).

A runbook lands only after two reviews with no blocking items, at the exact head they read.

## Blocking

- [ ] **The steps work.** Run the read-only steps, or a dry run of the others, and compare the output with what the runbook says. A step that can't run from where you are says so in the report.
- [ ] The frontmatter is complete and the `Last verified:` line is a date someone ran the runbook. A value of `never` is allowed and means the runbook is unverified, so the review says so.
- [ ] The sections follow the skeleton, and each command sits on its own line in a code block.
- [ ] Each failure path and the undo are present. A one-way door says so and says why.
- [ ] A helper script is a real file beside the runbook, and it matches what the steps call.
- [ ] The runbook names roles and durable components, never a session or a model run.
- [ ] A public runbook has no secrets, public IP addresses, private repository names, private host names, or tailnet details, and quotes no words of the operator verbatim. A private appendix is mentioned in one line and never linked.
- [ ] A planned tool or check reads as planned.
- [ ] The runbook doesn't expose a credential, spend money, widen anyone's authority, or weaken a stop or a review control.

## Non-blocking

- [ ] Paragraphs run 3 sentences or fewer, and each term keeps one name.
- [ ] A long procedure splits into linked runbooks.
- [ ] The outcome table covers the results a reader will see.

## Report

Report as a PR review, in this format:

```text
Runbook: <runbooks/role/procedure> in <owner/repo#N> at <sha>
Verdict: ready | changes needed
Blocking:
- <checklist item>: <file:line>, <what is wrong>, <the fix>
Non-blocking:
- <file:line>: <note>
Checked: <the steps run, the output compared, the sources read>
```

The verdict covers only that `sha`. A new commit needs a fresh review of at least the diff and of every item the diff could affect.
