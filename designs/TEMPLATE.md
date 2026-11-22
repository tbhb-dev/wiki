<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Design templates

Copy each skeleton into `designs/<slug>/`, replace the placeholders in angle brackets, and delete the guidance comments. [WRITING.md](WRITING.md) lists the rules each file follows, and [REVIEW.md](REVIEW.md) is what the reviewer checks.

## The README

The README gives an overview of the design. It doesn't contain decisions, open or decided, because those go in `decisions.md`. No heading in it may contain `Decisions`, `Open questions`, `Decided`, or `Already decided`. Keep it to about 250 lines and move the rest into appendices.

```markdown
---
type: Design Draft
title: <Title in plain words, at most 8 words>
description: <One sentence: what this designs and for whom.>
tags: [<topic>, <topic>]
status: draft            # draft | decided | superseded
owner: <durable role, never a session>
updated: "<YYYY-MM-DD>"
decisions_open: <N>      # the number of items under ## Open in decisions.md
---

# <Title>

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | <role> | <YYYY-MM-DD> | <N open decisions in [decisions.md](decisions.md), about M minutes> or Nothing |

## Summary

<At most 150 words, in 3 short paragraphs or fewer. Say what the problem is, what this proposes, and the one thing it changes for the operator. A reader who stops here knows the design's shape and its recommendation. Don't restate a question or an answer from decisions.md.>

## Proposal

<What exists when this is built, in about one page. Name the components and the role that runs each. Keep mechanics for the next section.>

## How it works

<Only the mechanics a reader needs to trust the proposal: subsections, numbered steps, tables, and diagrams. Move detail past about two pages into an appendix and link it.>

<Embed each render of a .d2 source in the design folder, and add no image that isn't one.>

![<What the diagram shows, in one sentence>](<name>.d2.svg)

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| <item> | <effect> | <mitigation, or the first build that answers it> | <proven: spike or PR / existing / new / unverified> |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | <deliverable> | <a check anyone can run> | <role> |

<No step waits on a meeting with the operator that only confirms documented behavior. Unknowns go into the first build that exercises them.>

## Appendices

- [Decisions](decisions.md): <N> open, then the decided record.
- [A. Evidence](A-evidence.md): <what the spikes, reads, and sources show, labeled proven, existing, new, or unverified>
- [B. Alternatives considered](B-alternatives.md): <the options not taken, and why>
- [C. <Topic>](C-<topic>.md): <detailed specification, schema, tables, or history>
```

## The decisions appendix

Every decision for the design goes in `decisions.md`, beside the README. Keep that exact name, with no letter. The open items come first, then the decided record. Each item keeps its number for life: an answered item moves from `## Open` to `## Decided` as it stands, and numbers aren't reused.

```markdown
---
type: Design Appendix
title: <Design title> decisions
description: The open and decided questions of the <design title> design.
tags: [<topic>, decisions, design]
status: draft
parent: README.md
---

# <Design title> decisions

Back to [the design](README.md).

## Open

<Only questions for the operator: a one-way door, a credential or a grant, a loosened trust boundary, or a standing rule of the operator's own. A role decides two-way doors and records them under Decided. If nothing is open, write "None." and keep the heading.>

### D1. <The question, in one line?>

- **Recommendation:** <the answer in one line>.
- **Why:** <one or two sentences>.
- **If yes:** <what happens next>.
- **If no:** <the alternative, and what it costs>.
- **Reversible:** <yes, and how / no, and why>.

<Repeat for D2, D3, and so on. the operator answers as "D1: yes".>

## Decided

<One entry for each decision, whether the operator or a role made it, newest first.>

### D2. <The question or choice, in one line>

- **Decided:** <YYYY-MM-DD>, by <the operator or a role>.
- **Rule:** <what now holds, in one or two sentences>.
- **Undo:** <how to reverse it, or "one-way door" and why>.
- **Record:** <the Decisions entry, PR, or commit that holds it>.
```

## A lettered appendix

Name each appendix `A-<topic>.md`, `B-<topic>.md`, and so on, one topic per file. Evidence goes in A, and alternatives in B. History of earlier revisions goes in an appendix, never in the README.

```markdown
---
type: Design Appendix
title: <Design title> <topic>
description: <One sentence on what this appendix holds.>
tags: [<topic>, design]
status: draft
parent: README.md
---

# <Design title> <topic>

Back to [the design](README.md).

<One topic, in as much detail as it needs.>
```

## Splitting off a private appendix

This repository is public, so a design starts here. Write the whole design here when nothing in it is private. When it needs private specifics, keep the bulk here and put only those specifics in a private appendix with the same slug.

The public README says that a private appendix exists, in one line under Appendices:

```markdown
- A private appendix holds the host-specific details for this design.
```

The public design never names the private repository, a private host, a tailnet detail, an internal URL, or a secret. Where it needs a private value, it says what the value is, and the appendix gives it.

## A diagram

Draw each diagram in D2 as `<name>.d2` in the design folder. The README embeds its render, `<name>.d2.svg` or `<name>.d2.png`, by relative path. Each render is three committed files beside the source: `<name>.d2.svg`, `<name>.d2.png`, and `<name>.d2.sha256`, the hash manifest of the source and both renders.

```d2
direction: right
author: Author role
reviewer: Reviewer role
author -> reviewer: design at an exact head
```

The `d2_diagrams` macro from tbhb-dev/rules renders and checks these files. The Bazel package that holds the `.d2` files calls it once:

```starlark
load("the operator_rules_d2//:defs.bzl", "d2_diagrams")

# `bazel run :diagrams` writes the renders beside each source.
# `:diagrams_test` fails when a render is stale, missing, or has no source.
d2_diagrams(
    name = "diagrams",
    data = [],  # imported .d2 files and local images, if any
)
```

### Planned wiring

The `d2_diagrams` call needs the `tbhb_rules_d2` module dependency in the repository's `MODULE.bazel`, which this repository doesn't have yet. The change that adds it also settles a `.gitattributes` entry that commits renders byte for byte, an exclusion of renders from the whitespace hooks, and where the call goes: one call in `designs/BUILD.bazel` covers every design, while a build file in each design folder splits them.

tbhb-dev/rules plans a check that fails a design README with a heading containing `Decisions`, `Open questions`, `Decided`, or `Already decided`, or with an image that doesn't come from a `.d2` source tracked in the same folder. That check isn't written yet.
