---
type: Content Template
title: Decision record template
description: The skeleton for a decision record in the adrs folder.
tags: [adrs, decisions, template]
status: draft
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Decision record template

Copy the skeleton into `adrs/NNNN-<slug>.md`, replace the placeholders in angle brackets, and delete any section that has nothing to say, except Context, Decision, Scope and Consequences.

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

## Scope

## Options considered

## Consequences

## Confirmation
````

## Writing rules

- The title states the decision, as in "Repository gates run as Bazel tests." A topic such as "Gates" is too vague.
- The Decision section states the rule in the present tense. It doesn't narrate who said what, and it never quotes anyone.
- The Scope section says what the decision covers, such as the systems, repositories, roles, or situations where the rule applies. It also says what the decision doesn't cover.
- The record defines every term, role, abbreviation, and label, or links it with a fully qualified link.
- Options considered is optional, with one line per rejected option and the reason it lost.
- Consequences list the costs as well as the benefits.
- Confirmation is optional, and it says how a reviewer or an agent checks that the rule is followed.
- A record states one decision and runs about one page. Detail and evidence go in linked documents.
- A record that replaces another names the replaced number in its Context. The replaced record gets `status: deprecated` and `superseded_by: <number>` in the same pull request.
- A record never names a host, the tailnet, a vault, an internal path, an account or a public IP address. It may name a private repository.
- Each paragraph goes on one line.
