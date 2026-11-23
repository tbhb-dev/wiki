---
type: Content Template
title: Decision record template
description: The skeleton for a decision record in the adrs folder.
tags: [adrs, decisions, template]
status: draft
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:c581062e61e4c416
    at: "2026-11-23T00:26:49Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Decision record template

Copy the skeleton into `adrs/NNNN-<slug>.md`, replace the placeholders in angle brackets, and delete any section that has nothing to say, except Context, Decision and Consequences.

## The skeleton

````markdown
---
type: Decision Record
title: <the decision, stated as a decision>
description: <one sentence, reused as the index entry>
tags: [<kebab-case>]
status: stable
decided: "<YYYY-MM-DD>"
---

# <title>

## Context

## Decision

## Options considered

## Consequences
````

## Writing rules

- The title states the decision, as in "Repository gates run as Bazel tests." A topic such as "Gates" is too vague.
- The Decision section states the rule in the present tense. It doesn't narrate who said what, and it never quotes anyone.
- Options considered is optional, with one line per rejected option and the reason it lost.
- Each paragraph goes on one line.
