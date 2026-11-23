---
type: Review Checklist
title: Decision record review checklist
description: What a reviewer checks in a pull request that adds or changes a decision record, with the source of each check.
tags: [adrs, review, checklist]
status: draft
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore Evolu Nygard Zimmermann -->

# Decision record review checklist

The reviewer didn't write or edit the record, and reads each changed record whole, together with the index entry and any record it replaces. Check each item against the files and the primary sources, not against the author's summary. The rules are in [README.md](README.md) and [TEMPLATE.md](TEMPLATE.md).

A record pull request reaches tbhb only after two reviews, Fable 5.2 and Codex Sol, with no blocking items, at the exact head each reviewer read.

Each item ends with the source of the check. A source named "tbhb ruling" is a standing ruling by tbhb, and the other sources are linked and summarized under Sources.

## Blocking

### Standing alone

- [ ] **Self-contained.** Every term, class, role, abbreviation, code, and label in the record is defined in the record or linked by a fully qualified link, and a bare "class R" fails. A stranger who has read only this repository can follow the record. Source: tbhb ruling self-contained, and [AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html).
- [ ] **Stranger-readable sections.** Context states the forces in neutral terms, Decision states the rule, Options considered names each rejected option with the reason it lost, and Consequences lists the good and the bad effects. Source: [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions) and [MADR](https://adr.github.io/madr/).
- [ ] **No hidden costs.** Consequences name at least one cost or risk, and a record that lists only benefits fails. Source: [Zimmermann on creating ADRs](https://ozimmer.ch/practices/2023/04/03/ADRCreation.html).
- [ ] **Decision stands without links.** The Decision section is clear without opening any linked document, and links only add evidence. Source: [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record).

### Faithful

- [ ] **The Decision matches the source Decisions entry.** The pull request cites the source entry, the reviewer opens that entry on a lab host and compares each sentence of the Decision with it. Source: tbhb ruling faithful.
- [ ] **Nothing invented, widened, or softened.** A rule the entry doesn't state is invented, a wider reach is widened, and a "should" for a "must" is softened. Each of the three fails. Source: tbhb ruling faithful.
- [ ] **Context and Options considered follow the entry too.** A reason, option, or consequence that the entry doesn't support fails and is removed. Source: [Zimmermann on creating ADRs](https://ozimmer.ch/practices/2023/04/03/ADRCreation.html), which names invented or wishful justification as an anti-pattern.

### Scope

- [ ] **The Scope section says what the decision covers.** It names the systems, repositories, roles, or situations where the rule applies. Source: tbhb ruling scope, and [AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html#scope-of-the-adr-process).
- [ ] **It also says what the decision does not cover.** At least one nearby case is named as outside the rule. Source: tbhb ruling scope.

### One decision

- [ ] **The record holds one decision.** A record that states two rules, or phases of one rule, is split into one record each. Source: [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions), [MADR](https://adr.github.io/madr/), and [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record).
- [ ] **The title states the decision.** A topic such as "Gates" fails, and "Repository gates run as Bazel tests" passes. Source: [MADR primer](https://ozimmer.ch/practices/2022/11/22/MADRTemplatePrimer.html) and [TEMPLATE.md](TEMPLATE.md).
- [ ] **The record is worth keeping.** The decision is hard to reverse, would surprise a reader without context, or has a real trade-off. A trivial or temporary choice fails. Source: [Fowler](https://www.martinfowler.com/bliki/ArchitectureDecisionRecord.html) and [Evolu](https://www.evolu.dev/blog/architecture-decision-records-for-coding-agents).

### Writing

- [ ] **The house writing rules hold.** Each sentence states one fact, each paragraph is on one line, each list item is a full sentence ending with a period, and nothing is meta exposition about the record itself. Source: tbhb ruling house writing rules, and [TEMPLATE.md](TEMPLATE.md).
- [ ] **Wording is observable.** The Decision names what a reader sees, runs, or can check, and avoids words such as "robust" or "appropriate". Source: tbhb ruling house writing rules.
- [ ] **The record is short.** It runs about one page, and detail, evidence, and measurements are linked instead of copied. Source: [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions) and [Fowler](https://www.martinfowler.com/bliki/ArchitectureDecisionRecord.html).
- [ ] **The Decision is a rule in the present tense.** It doesn't narrate who said what or quote anyone. Source: [TEMPLATE.md](TEMPLATE.md) and [README.md](README.md).

### Privacy

- [ ] **No host name, tailnet, vault name, internal path, account name, credential, or public IP address.** Role labels such as "the Linux build host" stand in for them. A private repository name is allowed. Source: tbhb ruling privacy, and [README.md](README.md).
- [ ] **No link to a private repository and no citation of a Decisions note.** Source: [README.md](README.md).

### Structure and filename

- [ ] **The filename is `NNNN-<slug>.md`.** The number is the next free one, zero-padded to four digits, and the slug is the kebab-case title. Source: [MADR](https://adr.github.io/madr/) and [adr-tools](https://github.com/npryce/adr-tools).
- [ ] **No number is reused or changed.** The number is unique among the merged records and the open pull requests, and a merged record isn't renamed. Source: [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions) and [Evolu](https://www.evolu.dev/blog/architecture-decision-records-for-coding-agents).
- [ ] **The frontmatter passes the schema.** It has `type: Decision Record`, `title`, `description`, `tags`, `status`, and `decided`, and the title matches the H1. Source: [the frontmatter schema](../docs/schemas/frontmatter.schema.json).
- [ ] **`decided` is the date of the decision.** It isn't the date of the record. Source: [README.md](README.md).
- [ ] **The index entry exists.** The Records list in README.md ends with the new entry, which carries the record's title and description. Source: [README.md](README.md).

### Duplicates and overlap

- [ ] **No record already states the rule.** The reviewer searches the existing records for the same rule or a conflicting one. A conflict is resolved by a replacing record, never by two live rules. Source: [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record).

### Status and supersession

- [ ] **The status is one of `draft`, `stable`, or `deprecated`.** Only a stable record binds, and a draft doesn't. Source: [the frontmatter schema](../docs/schemas/frontmatter.schema.json) and [BrainGrid](https://www.braingrid.ai/blog/architecture-decision-records-for-ai-coding-agents).
- [ ] **A replacing record names the replaced number.** The Context says which record it replaces and why. Source: [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions) and [adr-tools](https://github.com/npryce/adr-tools).
- [ ] **The old record changes in the same pull request.** Its status becomes `deprecated`, and `superseded_by` holds the replacing number. Its body stays as it was. Source: [AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html#review) and [Fowler](https://www.martinfowler.com/bliki/ArchitectureDecisionRecord.html).
- [ ] **A stable record isn't edited to change its rule.** An edit that changes the rule is a replacing record. Only a correction that keeps the rule, such as a typo or a broken link, edits in place. Source: [AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html#review) and [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record).

### Checkable by an agent

- [ ] **An agent that must follow the record can tell when it applies and whether it was followed.** The Scope names the trigger, and the Decision or the Confirmation section names the observable check. Source: [MADR](https://adr.github.io/madr/), [BrainGrid](https://www.braingrid.ai/blog/architecture-decision-records-for-ai-coding-agents), and [Evolu](https://www.evolu.dev/blog/architecture-decision-records-for-coding-agents).
- [ ] **The record doesn't depend on unstated knowledge.** An agent with this repository and the record can act without asking where a term comes from. Source: tbhb ruling self-contained.

## Non-blocking

- [ ] The title also names the problem it solves where that helps a search. Source: [MADR primer](https://ozimmer.ch/practices/2022/11/22/MADRTemplatePrimer.html).
- [ ] Options considered holds real alternatives at one level of abstraction, and no option exists only to make the winner look good. Source: [MADR primer](https://ozimmer.ch/practices/2022/11/22/MADRTemplatePrimer.html) and [Zimmermann on creating ADRs](https://ozimmer.ch/practices/2023/04/03/ADRCreation.html).
- [ ] Consequences say what would make the team revisit the decision. Source: [ecADR](https://ozimmer.ch/practices/2020/05/22/ADDefinitionOfDone.html) and [Fowler](https://www.martinfowler.com/bliki/ArchitectureDecisionRecord.html).
- [ ] The record states how confident the decision is where the confidence is low. Source: [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record).
- [ ] Review comments name the file, the line, and the fix, and skip wording preferences that don't change meaning. Source: [Zimmermann on reviewing ADRs](https://ozimmer.ch/practices/2023/04/05/ADRReview.html).

## Report

Report as a PR review, in this format:

```text
Record: <adrs/NNNN-slug> in <owner/repo#N> at <sha>
Verdict: ready for tbhb | changes needed
Blocking:
- <checklist item>: <file:line>, <what is wrong>, <the fix>
Non-blocking:
- <file:line>: <note>
Checked: <the source Decisions entry compared, the records searched for overlap, the number checked for reuse>
```

The verdict covers only that `sha`. A new commit needs a fresh review of at least the diff and of every item the diff could affect.

## Sources

- [Nygard, Documenting architecture decisions](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions) fixed the five parts, the neutral Context, the one-to-two-page length, the rule that numbers are never reused, and the rule that a reversed decision stays in place marked superseded. It produced the sections, one-page, numbering, and supersession items.
- [MADR](https://adr.github.io/madr/) fixed the single-decision rule, the `NNNN-title-with-dashes.md` filename, and the optional Confirmation section. It produced the one-decision, filename, and checkable items, and it added a Confirmation section to the template.
- [Zimmermann, MADR template primer](https://ozimmer.ch/practices/2022/11/22/MADRTemplatePrimer.html) fixed the title that names problem and solution, options at one abstraction level, and consequences that include costs. It produced the title and options items, and it confirmed the template's section order with no change.
- [Zimmermann, how to create ADRs](https://ozimmer.ch/practices/2023/04/03/ADRCreation.html) listed the anti-patterns Fairy Tale, Free Lunch Coupon, Dummy Alternative, and Mega-ADR. It produced the no-hidden-costs item, the no-invented-justification item, and the short-record item.
- [Zimmermann, how to review ADRs](https://ozimmer.ch/practices/2023/04/05/ADRReview.html) listed a seven-question review checklist and the review anti-patterns Pass Through and Copy Edit. It confirmed the open-the-whole-record intro, and it produced the non-blocking item on comments.
- [Zimmermann, ecADR](https://ozimmer.ch/practices/2020/05/22/ADDefinitionOfDone.html) defines done by evidence, criteria, agreement, documentation, and a review plan. It produced the revisit item. Agreement is the two-review gate, and the checklist adds no evidence criterion because the source Decisions entry is the evidence.
- [AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html) says an accepted record is immutable, a new record supersedes it, and the old state changes to Superseded. It also names what counts as architecturally significant. It produced the supersession items and the scope item.
- [Azure Well-Architected, maintain an ADR](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) says to log phases as separate records, to state confidence, and to keep the decision clear without the linked material. It produced the one-decision, standalone, and confidence items.
- [Google Cloud, architecture decision records](https://docs.cloud.google.com/architecture/architecture-decision-records) says to keep records near the code, to record rejected alternatives, and to keep the previous decision when one changes. It confirmed the in-repo location and the options section, with no change.
- [Fowler, Architecture Decision Record](https://www.martinfowler.com/bliki/ArchitectureDecisionRecord.html) says to keep a record to about one page, to state confidence and revisit triggers, and to change a superseded record's status only. It produced the length, revisit, and old-record items.
- [adr-tools](https://github.com/npryce/adr-tools) marks the replaced record when a new one supersedes it. It produced the same-pull-request rule for the old record.
- [adr.github.io](https://adr.github.io/) defines a record as one architectural decision with its rationale and lists the Y-statement form. It confirmed the one-decision rule. The Y-statement was not adopted, because the Context, Decision, and Consequences sections already cover its clauses.
- [Joel Parker Henderson, architecture decision record](https://github.com/joelparkerhenderson/architecture-decision-record) lists rationale, specific, timestamped, and immutable as the traits of a good record, and says to skip small or temporary decisions. It produced the worth-keeping item and confirmed the `decided` date, with no change.
- [ThoughtWorks Radar, lightweight ADRs](https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records) rated the technique Adopt and said to keep records in source control. It confirmed the in-repo practice, with no change.
- [Evolu, ADRs for coding agents](https://www.evolu.dev/blog/architecture-decision-records-for-coding-agents) has agents read records before designing, write new ones as Proposed, and stop when a task conflicts with an accepted record. It warns that parallel sessions collide on the next number. It produced the worth-keeping and agent-checkable items, and it confirmed the README's rule that the later pull request rebases and takes the next number.
- [BrainGrid, ADRs for AI coding agents](https://www.braingrid.ai/blog/architecture-decision-records-for-ai-coding-agents) says to use few states so an agent can tell whether a record binds, to give explicit triggers, and to point agents at the folder from their instruction file. It produced the status and agent-checkable items.
- [AGENTS.md](https://agents.md/) says agents read the nearest instruction file at the repository root. It confirmed that the discovery pointer belongs in the repository's `AGENTS.md`, with no change to the records themselves.
- [The frontmatter schema](../docs/schemas/frontmatter.schema.json) fixes the Decision Record fields, the statuses `draft`, `stable`, and `deprecated`, and `superseded_by` on deprecated records. It produced the frontmatter and status items.
- tbhb ruling self-contained, faithful, and scope names the three checks the reviewer applies to every record. tbhb ruling privacy allows a private repository name and bars the other internal names. tbhb ruling house writing rules sets the sentence, paragraph, and list rules.
- tbhb ruling review criteria come from the research set this file's method. No source here sets a word cap for a record, so the template keeps the length guidance at about one page and adds no cap.
