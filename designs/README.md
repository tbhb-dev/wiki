<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Designs

Each design has its own directory, `designs/<slug>/`. Its `README.md` gives an overview of the design, `decisions.md` records every decision, and lettered appendices give the detail. Each design's frontmatter records its status.

## A design folder

| File | Contents |
| --- | --- |
| `README.md` | An overview with the status table, summary, proposal, mechanics, risks, plan, and links to every appendix. Open and decided items both go in `decisions.md` instead. |
| `decisions.md` | Every decision: the open ones under `## Open`, then the decided record under `## Decided`. |
| `A-<topic>.md`, `B-<topic>.md` | Appendices, one topic each, such as evidence, alternatives, or history. |
| `<name>.d2` | The D2 source of a diagram the README shows. |
| `<name>.d2.svg`, `<name>.d2.png`, `<name>.d2.sha256` | The committed renders of that source and their hash manifest. |

## Public designs and private appendices

A design starts in this public repository. A design that needs private specifics keeps the bulk here and puts only those specifics in a private appendix with the same slug. [TEMPLATE.md](TEMPLATE.md) shows the wording.

## Writing and reviewing a design

- [TEMPLATE.md](TEMPLATE.md) gives the skeletons for the README, `decisions.md`, an appendix, and a diagram.
- [WRITING.md](WRITING.md) is the author's checklist before review.
- [REVIEW.md](REVIEW.md) is the reviewer's checklist and report format.

A design reaches the operator only after a reviewer who didn't write it clears it of blocking items at the exact head it reviewed. Designs written before these conventions keep their earlier layout until someone next revises them.

## Planned checks

- A tbhb-dev/rules check is to fail a design README with a heading containing `Decisions`, `Open questions`, `Decided`, or `Already decided`, or with an image that isn't a render of a `.d2` source tracked in the same folder. It isn't written yet.
- A planned tbhb-dev/rules target generates an index of designs from each design's frontmatter, to replace this page's list. Until it exists, list the directory to see the designs.
