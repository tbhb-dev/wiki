<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Design review checklist

The reviewer didn't write or edit the design, and reads the whole folder: the README, `decisions.md`, every appendix, and every diagram source. Check each item against the files and primary sources, not against the author's summary. The rules are in [README.md](README.md), [TEMPLATE.md](TEMPLATE.md), and [WRITING.md](WRITING.md).

A design reaches the operator only after a review with no blocking items, at the exact head that review read.

## Reviewer models

A design takes two reviews, each from a model that didn't write it. Both come from Codex or Antigravity at equal capability: Codex at its top model on high effort, or Antigravity's top Gemini model. The author runs on Opus 5.5 at medium effort through the launcher, so a Claude model never reviews its own family's design. [../models/README.md](../models/README.md) has the roster.

## Blocking

### Structure

- [ ] The frontmatter is complete: `type`, `title`, `description`, `tags`, `status`, `owner`, `updated`, and `decisions_open`.
- [ ] The status table comes first, then a Summary of at most 150 words, then Proposal, How it works, Risks and unknowns, Plan, and Appendices.
- [ ] A reader who stops after the Summary knows the design's shape and its recommendation.
- [ ] The README is about 250 lines or fewer. Detail, evidence, and history sit in appendices.
- [ ] Each appendix and `decisions.md` has `type: Design Appendix` frontmatter and a link back to the README. The README links each of them, and every link resolves.
- [ ] Lettered appendices use `A-<topic>.md`, `B-<topic>.md`, and so on. The decisions file is exactly `decisions.md`, with no letter.

### Decisions

- [ ] **The README has no decisions section.** No heading in it contains `Decisions`, `Open questions`, `Decided`, or `Already decided`. Its text states no question for the operator and no answered decision. It may give the count and link `decisions.md`.
- [ ] Lettered appendices don't hold open questions or decision records, because those belong in `decisions.md`.
- [ ] **`decisions.md` exists, open first.** `## Open` comes before `## Decided`. An empty Open section says `None.` and keeps its heading.
- [ ] The frontmatter `decisions_open` and the status table both equal the number of items under `## Open`.
- [ ] Each open item has Recommendation, Why, If yes, If no, and Reversible, and takes a one-line answer such as `D1: yes`.
- [ ] Each open item is a question for the operator. It's a one-way door, a credential, a grant, a loosened trust boundary, or a standing rule of the operator's own. A two-way door that a role can decide goes under `## Decided`.
- [ ] Search the decision records for each open item. If the operator already answered one, move it to `## Decided`.
- [ ] Run the read-only checks an open item depends on, and record the result in the item. An item can't wait on a check nobody ran.
- [ ] Each decided entry gives a date, who decided, the rule, how to undo it, and a record. Open each record and confirm it says what the entry claims.
- [ ] **Hold a design PR that has open items.** The PR body says `Open decisions: N` in its Needs human judgment section, and N equals `decisions_open`. When N exceeds 0, the reviewer doesn't approve and nobody merges the PR until the operator answers. The answer moves the item to `## Decided`. The PR gets a new head, and the reviewer reviews that head again.
- [ ] **A merged design becomes tracked work.** When the design merges, its plan steps go into the relevant tracker in the tracker for its area, or into a new one. Each step has an owner and an evidence column. The merger or the design's owner does this.

### Diagrams

- [ ] **Every image is a D2 render with a tracked source.** Each image in the README is `<name>.d2.svg` or `<name>.d2.png`, linked by relative path, and `<name>.d2` sits in git in the same folder. Screenshots, hand-drawn images, and remote URLs fail this item.
- [ ] **Render files are current.** Each source has its `.d2.svg`, `.d2.png`, and `.d2.sha256` beside it. The manifest's hashes match `shasum -a 256` of the three files, and the SVG matches a fresh render where the D2 target runs.

### Substance

- [ ] The design refers to durable roles and components, never to a specific session or a long-running coordinator. For each actor, it says what runs it and where it keeps its state.
- [ ] Each factual claim has a label (proven, existing, new, or unverified) and a citation in appendix A. Spot-check at least three against their sources.
- [ ] The design agrees with the decision records, merged code, and other current designs. Where it keeps an older claim, it says which source takes precedence.
- [ ] A planned check, target, or service reads as planned, never as something that exists today.
- [ ] Each plan step has a `Done when` check anyone can run, and a role as owner. A step that waits on a meeting with the operator only to confirm documented behavior fails this item.
- [ ] Check the design against the standing rules. It doesn't expose a credential or spend money. It doesn't propose API keys or per-token billing, and it doesn't widen anyone's authority or weaken a stop or a review control.

### Privacy

- [ ] The change has no secrets and no public IP addresses.
- [ ] It quotes no words of the operator verbatim.
- [ ] A public design names no private repository, host, tailnet detail, or internal URL. When a private appendix exists, the design says so in one line and links nothing private.

## Non-blocking

- [ ] Paragraphs run 3 sentences or fewer. Lists of 3 or more items are bullets, not inline runs.
- [ ] Each term gets one definition at first use and keeps the same name everywhere.
- [ ] Paths sit in backticks, in plain words without filler.
- [ ] Each risk row gives a mitigation, or the first build that answers the question.
- [ ] A diagram shows something the prose would take longer to say.

## Report

Report as a PR review, in this format:

```text
Design: <designs/slug> in <owner/repo#N> at <sha>
Verdict: ready for the operator | changes needed
Blocking:
- <checklist item>: <file:line>, <what is wrong>, <the fix>
Non-blocking:
- <file:line>: <note>
Checked: <the decision records searched, the sources spot-checked, the render hashes compared>
```

The verdict covers only that `sha`. A new commit needs a fresh review of at least the diff and of every item the diff could affect.
