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

Copy the skeleton into `adrs/NNNN-<slug>.md` and replace the placeholders in angle brackets. Keep Context, Decision, Consequences, and a statement of the rule's scope. Delete any optional section that has nothing to say.

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
- The description is one sentence. Reuse it in the index entry.
- The Decision section states the rule in the present tense. It doesn't narrate who said what, and it never quotes anyone.
- Use the Scope heading to state where the decision applies and where it ends. A clear statement of scope in Context still gives readers the boundaries they need.
- The record defines every term, class, role, abbreviation, code, and label it uses. A fully qualified link to a record that defines the term counts as defining it.
- Options considered is optional, with one line per rejected option and the reason it lost.
- Consequences list the costs as well as the benefits.
- Confirmation is optional, and it says how a reviewer or an agent checks that the rule is followed.
- A record states one decision and runs about one page. The word caps are 150 for Context, 120 for Decision, 150 for Consequences, and 450 for the whole record. Detail and evidence go in linked documents.
- A record that replaces another names the replaced number in its Context. The replaced record gets `status: deprecated` and `superseded_by` as an integer without leading zeros in the same pull request. Use `superseded_by: 7` for replacement record `0007`.
- A record never names a host, the tailnet, a vault, an internal path, an account or a public IP address. It may name a private repository.
- Keep generated attribution text, including the `generated` field and authorship trailers, out of the record.
- Each paragraph goes on one line.
- The file ends with exactly one newline.
