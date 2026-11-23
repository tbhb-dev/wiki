---
type: Content Template
title: Model roster entry template
description: The skeleton for a model roster entry.
tags: [models, template]
status: draft
generated:
  - role: generated
    harness: claude-code:2.1.322
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:50691a4df0360088
    at: "2026-11-13T03:58:50Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- The placeholders in angle brackets are template text. -->
<!-- rumdl-disable MD033 -->

# Model roster entry template

The page is short. It has recommendations and standing rulings, and no record of when or by whom a finding came.

## Job row

Add a row to the Models table in [README.md](README.md), one row for each job.

```markdown
| <job> | <models and harnesses that fit, with unverified beside any claim that lacks support> | <what not to use for the job> |
```

## Observation line

Add a line under the model's heading in [OBSERVATIONS.md](OBSERVATIONS.md).

```markdown
- <Succeeded, Failed, or Mixed> at <task type>. <One sentence on what it did, and the limit of the test.>
```
