---
type: Review Checklist
title: Model roster review checklist
description: What a reviewer checks in a model roster change.
tags: [models, review, checklist]
status: draft
generated:
  - role: generated
    harness: claude-code:2.1.322
    model: anthropic:claude-opus-5-5
    effort: medium
    surface: cli
    session: sha-256-64:50691a4df0360088
    subagent: sha-256-64:5b7fc139aba28d0b
    at: "2026-11-13T03:40:01Z"
  - role: generated
    harness: claude-code:2.1.322
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:50691a4df0360088
    at: "2026-11-13T03:40:01Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model roster review checklist

The reviewer didn't write the change and reads the whole diff against [README.md](README.md), [TEMPLATE.md](TEMPLATE.md), and [WRITING.md](WRITING.md).

## Blocking

- [ ] The README is short and has no dates, run ids, session names, sample counts, or timestamps.
- [ ] Every recommendation rests on a standing ruling or an OBSERVATIONS.md line, or says `unverified`.
- [ ] No row contradicts a standing ruling.
- [ ] Rows match their formats in TEMPLATE.md.
- [ ] The change has no secrets, public IP addresses, host names, tailnet details, internal paths, private repository names, PR numbers from private repositories, account addresses, or session names.
- [ ] Any API spend cites @tbhb's approval.

## Non-blocking

- [ ] Each model and task type has one name everywhere.
- [ ] Lines are one or two plain sentences.

## Report

Report as a PR review in this format:

```text
Change: <owner/repo#N> at <sha>
Verdict: ready to merge | changes needed
Blocking:
- <checklist item>: <file:line>, <what is wrong>, <the fix>
Non-blocking:
- <file:line>: <note>
Checked: <what was opened, and what it showed>
```

The verdict covers only that `sha`. A new commit needs a fresh review of at least the diff.
