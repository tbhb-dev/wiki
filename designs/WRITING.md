<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Design writer checklist

Follow this list before asking for review of a design. Use the skeletons in [TEMPLATE.md](TEMPLATE.md). The reviewer checks the same ground with [REVIEW.md](REVIEW.md).

## Folder

- [ ] The design has its own folder, `designs/<slug>/`. It has `README.md`, `decisions.md`, and lettered appendices `A-<topic>.md`, `B-<topic>.md`, and so on.
- [ ] `decisions.md` keeps that exact name and takes no letter.
- [ ] Each file follows its skeleton in TEMPLATE.md and stays well under 30,000 bytes.

## The README

- [ ] The README gives an overview of the design and nothing more. Questions and answers belong in `decisions.md`.
- [ ] No heading in the README contains `Decisions`, `Open questions`, `Decided`, or `Already decided`.
- [ ] The README has `type: Design Draft` frontmatter. Each lettered appendix and `decisions.md` has `type: Design Appendix` and `parent: README.md`.
- [ ] The frontmatter is complete, and `decisions_open` equals the number of items under `## Open` in `decisions.md`.
- [ ] The status table comes first, then a Summary of at most 150 words, then Proposal, How it works, Risks and unknowns, Plan, and Appendices.
- [ ] The README runs about 250 lines or fewer. Detail, evidence, and history go in appendices.
- [ ] The README links every appendix and `decisions.md`, and each link resolves.

## The decisions appendix

- [ ] `## Open` comes first, then `## Decided`. If nothing is open, the Open section says `None.` and keeps its heading.
- [ ] Each open item has Recommendation, Why, If yes, If no, and Reversible, and the operator can answer it in one line, such as `D1: yes`.
- [ ] An open item asks the operator only about a one-way door, a credential, a grant, a loosened trust boundary, or a standing rule of the operator's own. A role decides two-way doors and records them under `## Decided`.
- [ ] Lettered appendices don't hold questions or decisions, because those go in `decisions.md`.
- [ ] Each decided entry gives the date, who decided, the rule, how to undo it, and the record that holds it.
- [ ] An answered item moves from Open to Decided and keeps its number. Numbers aren't reused.

## Diagrams

- [ ] The README has a diagram wherever a picture explains faster than prose. A request flow or a trust boundary is a good candidate.
- [ ] Each image in the README is a render of a `.d2` source in the same folder, embedded by relative path as `<name>.d2.svg` or `<name>.d2.png`.
- [ ] After each edit to a source, the author renders it again and commits the source with its `.d2.svg`, `.d2.png`, and `.d2.sha256`.
- [ ] Screenshots, hand-drawn images, and remote image URLs stay out of the README.

## Substance

- [ ] The design refers to durable roles and components, never to a specific session or a long-running coordinator. For each actor, it says what runs it and where it keeps its state.
- [ ] Each factual claim has a label (proven, existing, new, or unverified), with its citation in appendix A.
- [ ] A planned check, target, or service reads as planned.
- [ ] Each plan step has a `Done when` check anyone can run, and a role as owner. A step never waits on a meeting with the operator that only confirms documented behavior.

## Privacy and style

- [ ] The change has no secrets and no public IP addresses, and quotes no words of the operator verbatim.
- [ ] The public design names no private repository, host, tailnet detail, or internal URL. Private specifics go in a private appendix with the same slug, and the design says in one line that one exists.
- [ ] Each Markdown paragraph is on one line and runs 3 sentences or fewer. Headings use sentence case.
- [ ] Each file spells out an acronym at its first use there, and lists use the Oxford comma.
- [ ] `vale`, `rumdl check`, and `cspell` pass on the changed files, or `bazelisk test //...` passes.

## Review and the PR

- [ ] Before the design reaches the operator, a reviewer who didn't write it must clear it of blocking items at the exact head it reviewed.
- [ ] The PR body fills every section of the PR template.
- [ ] The PR body says `Open decisions: N` in its Needs human judgment section, where N is the number of items under `## Open`. Nobody merges a design PR that has an open item until the operator answers it.
- [ ] When the design merges, its plan steps go into the tracker for its area, or into a new tracker. Each step has an owner and an evidence column.
