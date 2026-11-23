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

The review tier sets the initial review count. A decision record pull request needs two fresh runs from different model families, and a fix needs one fresh run on the fix diff. The operator approves the reviewed head once all blocking findings are resolved. See [ADR 0001](https://github.com/tbhb-dev/wiki/blob/22d528cbb73d640ff1291e3a5dd1e9f9ae89a791/adrs/0001-the-changed-paths-and-content-set-the-review-tier.md).

Count each run only for the commits for which its reviewer qualifies under the provider rule. This also applies to the sole delta reviewer for a fix. See [ADR 0002](https://github.com/tbhb-dev/wiki/blob/22d528cbb73d640ff1291e3a5dd1e9f9ae89a791/adrs/0002-reviewers-come-from-a-different-provider-from-the-builder.md).

Each item ends with the source of the check. The Sources section summarizes the references and the operator's standing rulings.

## BLOCK

Assign BLOCK when ordinary use without an adversary produces a wrong result, destroys data, or breaks a workflow. For disclosures, only a credential or secret stops review. See the operator's disclosure ruling and [ADR 0007](https://github.com/tbhb-dev/wiki/blob/22d528cbb73d640ff1291e3a5dd1e9f9ae89a791/adrs/0007-a-review-finding-blocks-when-ordinary-use-breaks.md).

Deliberate evasion of a scanner, linter, or other check also blocks. Fix the underlying issue or use the check's reviewed allowlist. Redaction placeholders can stand in for information that must be withheld. See [ADR 0008](https://github.com/tbhb-dev/wiki/blob/22d528cbb73d640ff1291e3a5dd1e9f9ae89a791/adrs/0008-deliberately-evading-a-check-blocks-review.md).

### Standing alone

- [ ] **Self-contained.** The record defines every term, class, role, abbreviation, code, and label it uses. A fully qualified link to a record that defines the term counts as defining it. Source: the operator's self-contained ruling and [TEMPLATE.md](TEMPLATE.md).
- [ ] **The Decision states the complete rule.** Linked documents can supply evidence and definitions. The decision itself appears in the record. Source: [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) and the operator's self-contained ruling.

### Faithful

- [ ] **The Decision matches the source Decisions entry.** The pull request cites the Decisions entry. The reviewer opens it on a lab host and compares each sentence of the Decision with it. Source: the operator's fidelity ruling and [README.md](README.md).
- [ ] **The record preserves the rule, its reach, and its force.** Flag content that is wrong or contradicts the Decisions entry, including an invented rule, a wider reach, or a "should" in place of a "must." Source: the operator's fidelity ruling.
- [ ] **The rest of the record follows the Decisions entry.** Flag a reason, option, or consequence that the Decisions entry does not support. Source: the operator's fidelity ruling and [Zimmermann on creating ADRs](https://ozimmer.ch/practices/2023/04/03/ADRCreation.html).

### Scope

- [ ] **The record makes the rule's boundaries clear.** It names the systems, repositories, roles, or situations where the rule applies and where it ends. Block only if missing or incorrect scope would cause the reader to misapply the rule. Clear boundaries in Context qualify, as in [ADR 0029](https://github.com/tbhb-dev/wiki/blob/41816a4048b5ceab461a011440bf4ec44380d499/adrs/0029-wording-only-follow-ups-join-the-next-needed-pull-request.md). See the operator's scope ruling and [AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html#scope-of-the-adr-process).

### One decision

- [ ] **The record holds one decision.** A record that states two rules, or phases of one rule, is split into one record each. Source: [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions), [MADR](https://adr.github.io/madr/), and [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record).

### Privacy

- [ ] **The record contains no credential or secret.** Source: the operator's disclosure ruling and [README.md](README.md).

### Structure and filename

- [ ] **The filename is `NNNN-<slug>.md`.** The number is the next free one, zero-padded to four digits, and the slug is the kebab-case title. Source: [MADR](https://adr.github.io/madr/) and [adr-tools](https://github.com/npryce/adr-tools).
- [ ] **Record numbers are unique.** When two open pull requests claim the same number, the later one rebases and takes the next free number. A merged record keeps its name except when fixing a duplicate number, in which case the later file takes the next free number. Source: [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions), [Evolu](https://www.evolu.dev/blog/architecture-decision-records-for-coding-agents), and [README.md](README.md).
- [ ] **The frontmatter passes the schema.** It has `type: Decision Record`, `title`, `description`, `tags`, `status`, and `decided`, and the title matches the H1. Source: [the frontmatter schema](../docs/schemas/frontmatter.schema.json).
- [ ] **`decided` gives the date the decision was made.** Source: [README.md](README.md) and the [frontmatter schema](../docs/schemas/frontmatter.schema.json).
- [ ] **The index entry exists.** The Records list in README.md ends with the new entry, which carries the record's title and description. Source: [README.md](README.md).

### Duplicates and overlap

- [ ] **No record already states the rule.** The reviewer searches the existing records for the same rule or a conflicting one. A conflict is resolved by a replacing record, never by two live rules. Source: [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record).

### Status and supersession

- [ ] **The status is one of `draft`, `stable`, or `deprecated`.** Only a stable record binds, and a draft doesn't. Source: [the frontmatter schema](../docs/schemas/frontmatter.schema.json) and [BrainGrid](https://www.braingrid.ai/blog/architecture-decision-records-for-ai-coding-agents).
- [ ] **A replacing record names the replaced number.** The Context says which record it replaces and why. Source: [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions) and [adr-tools](https://github.com/npryce/adr-tools).
- [ ] **The old record changes in the same pull request.** Its status becomes `deprecated`, and `superseded_by` holds the replacing number as an integer without leading zeros. For replacement record `0007`, use `superseded_by: 7`. Its body stays as it was. Source: [adr-tools](https://github.com/npryce/adr-tools), [Fowler](https://www.martinfowler.com/bliki/ArchitectureDecisionRecord.html), and the [frontmatter schema](../docs/schemas/frontmatter.schema.json).
- [ ] **A stable record isn't edited to change its rule.** An edit that changes the rule is a replacing record. Only a correction that keeps the rule, such as a typo or a broken link, edits in place. Source: [AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html#review) and [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record).

### Checkable by an agent

- [ ] **An agent that must follow the record can tell when it applies and whether it was followed.** The record names the trigger wherever it states the scope. The Decision or the Confirmation section names the observable check. See [MADR](https://adr.github.io/madr/), [BrainGrid](https://www.braingrid.ai/blog/architecture-decision-records-for-ai-coding-agents), and [Evolu](https://www.evolu.dev/blog/architecture-decision-records-for-coding-agents).
- [ ] **The record supplies the knowledge needed to act.** An agent can find every required definition in the record or in a record it links directly. Source: the operator's self-contained ruling.

## TECH-DEBT

### Writing

- [ ] **Stranger-readable sections.** Context states the forces in neutral terms, Decision states the rule, and Consequences lists the good and the bad effects. Options considered is optional. If present, it names each rejected option with the reason it lost. Source: [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions), [MADR](https://adr.github.io/madr/), and [TEMPLATE.md](TEMPLATE.md).
- [ ] **No hidden costs.** Consequences name at least one cost or risk. Source: [Zimmermann on creating ADRs](https://ozimmer.ch/practices/2023/04/03/ADRCreation.html).
- [ ] **The record uses a Scope heading for its boundaries.** When the boundaries are clear under Context or another heading, flag placement as a writing issue. See [TEMPLATE.md](TEMPLATE.md).
- [ ] **The record names a nearby case outside the rule.** See the operator's scope ruling.
- [ ] **The title states the decision.** "Repository gates run as Bazel tests" states the rule. Source: [MADR primer](https://ozimmer.ch/practices/2022/11/22/MADRTemplatePrimer.html) and [TEMPLATE.md](TEMPLATE.md).
- [ ] **The record is worth keeping.** The decision is hard to reverse, would surprise a reader without context, or has a real trade-off. Source: [Fowler](https://www.martinfowler.com/bliki/ArchitectureDecisionRecord.html), [Henderson](https://github.com/joelparkerhenderson/architecture-decision-record), and [Evolu](https://www.evolu.dev/blog/architecture-decision-records-for-coding-agents).
- [ ] **The house writing rules hold.** Each sentence states one fact, each paragraph is on one line, and each list item is a full sentence ending with a period. Keep the prose focused on the decision. Source: the operator's house writing ruling and [TEMPLATE.md](TEMPLATE.md).
- [ ] **Wording is observable.** The Decision names what a reader sees, runs, or can check. Source: the operator's house writing ruling.
- [ ] **The record is short.** Aim for about one page within the word caps in [TEMPLATE.md](TEMPLATE.md). Link to detail, evidence, and measurements. Source: [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions), [Fowler](https://www.martinfowler.com/bliki/ArchitectureDecisionRecord.html), and [TEMPLATE.md](TEMPLATE.md).
- [ ] **The Decision states the rule in the present tense.** Keep quotations and accounts of who said what out of the record. Source: [TEMPLATE.md](TEMPLATE.md) and [README.md](README.md).
- [ ] **The description is one sentence.** Reuse it in the index entry. See [TEMPLATE.md](TEMPLATE.md).
- [ ] **The file ends with exactly one newline.** See [TEMPLATE.md](TEMPLATE.md) and [the EditorConfig rules](../.editorconfig).
- [ ] **The record omits generated attribution text.** Keep the `generated` field and authorship trailers out of the record. See [TEMPLATE.md](TEMPLATE.md).

### Privacy

- [ ] **Keep links to private repositories and Decisions entry citations out of the record.** Source: [README.md](README.md).

### Other checks

- [ ] The title also names the problem it solves where that helps a search. Source: [MADR primer](https://ozimmer.ch/practices/2022/11/22/MADRTemplatePrimer.html).
- [ ] If included, Options considered holds real alternatives at one level of abstraction. Source: [MADR primer](https://ozimmer.ch/practices/2022/11/22/MADRTemplatePrimer.html) and [Zimmermann on creating ADRs](https://ozimmer.ch/practices/2023/04/03/ADRCreation.html).
- [ ] Consequences say what would make the team revisit the decision. Source: [ecADR](https://ozimmer.ch/practices/2020/05/22/ADDefinitionOfDone.html) and [Fowler](https://www.martinfowler.com/bliki/ArchitectureDecisionRecord.html).
- [ ] The record states how confident the decision is where the confidence is low. Source: [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record).
- [ ] Review comments name the file, the line, and the fix, and skip wording preferences that don't change meaning. Source: [Zimmermann on reviewing ADRs](https://ozimmer.ch/practices/2023/04/05/ADRReview.html).

## SECURITY-DEBT

- [ ] **Role labels replace identifying details.** Use role labels such as "the Linux build host" for host names, tailnet details, vault names, internal paths, account names, and public IP addresses. A private repository name is allowed. See the operator's disclosure ruling and [README.md](README.md).

## Report

Report as a PR review, in this format:

```text
Record: <adrs/NNNN-slug> in <owner/repo#N> at <sha>
Tier: <review tier>
Run ID: <review run id>
Provider: <model provider>
Verdict: GO | HOLD
BLOCK:
- <checklist item>: <file:line>, <what is wrong>, <the fix>
SECURITY-DEBT:
- <file:line>: <note>
TECH-DEBT:
- <file:line>: <note>
Checked: <the source Decisions entry compared, the records searched for overlap, the number checked for reuse>
```

The verdict covers only the reported head. See [ADR 0004](https://github.com/tbhb-dev/wiki/blob/22d528cbb73d640ff1291e3a5dd1e9f9ae89a791/adrs/0004-review-runs-publish-verdicts-on-the-pull-request.md). The fresh delta run after a fix checks the fix diff and each checklist item the diff could affect. Count the delta run only if its reviewer qualifies for the fix commits under the provider rule above.

A history rewrite requires fresh review at the applicable tier. See [ADR 0021](https://github.com/tbhb-dev/wiki/blob/41816a4048b5ceab461a011440bf4ec44380d499/adrs/0021-authors-may-update-and-curate-pull-request-branches.md). The review worker can reuse approval for a rebuilt replacement whose Git tree matches the approved tree. Tree identity stands in for the reviewer evidence for the exact head and the approver's read of the full diff. The earlier verdict stays attached to the commit the reviewer read, and the tree match is evidence for a new approval on the replacement's head. At that head, the `conclusion` check, verified signatures and rejection of the opener's root identity still apply, and the worker holds a replacement that changes the review template. See [ADR 0010](https://github.com/tbhb-dev/wiki/blob/e43bbdc5ebf3faa68b39dd294f7eade44841f80a/adrs/0010-a-rebuilt-pull-request-can-reuse-approval-for-an-identical-tree.md).

## Sources

- [Nygard](https://www.cognitect.com/blog/2011/11/15/documenting-architecture-decisions) describes the parts of a record, a neutral Context, one decision per record, and a length of one to two pages. He calls for unique numbers and keeping a superseded record with its replacement identified.
- [MADR](https://adr.github.io/madr/) informs the section layout, single-decision rule, numbered filename, and optional Confirmation section.
- The [MADR primer](https://ozimmer.ch/practices/2022/11/22/MADRTemplatePrimer.html) describes a title that names the problem and solution and options at one level of abstraction.
- [Zimmermann on creating ADRs](https://ozimmer.ch/practices/2023/04/03/ADRCreation.html) warns against invented justifications, hidden costs, and options included solely to justify the chosen option.
- [Zimmermann on reviewing ADRs](https://ozimmer.ch/practices/2023/04/05/ADRReview.html) describes review pitfalls, including copy edits that neglect the decision itself. It informs the item on review comments.
- [ecADR](https://ozimmer.ch/practices/2020/05/22/ADDefinitionOfDone.html) includes a review plan in its definition of done. It informs the check for a reason to revisit a decision.
- [AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html) describes the scope of the ADR process and says a new record supersedes an accepted record, which keeps its content and changes state.
- [Azure](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record) recommends logging phases as separate records, resolving conflicting guidance, stating confidence, and keeping the decision clear without linked material. The operator's self-contained ruling permits links that supply definitions.
- [Google Cloud, architecture decision records](https://docs.cloud.google.com/architecture/architecture-decision-records) says to keep records near the code, to record rejected alternatives, and to keep the previous decision when one changes. It confirmed the in-repo location and the options section, with no change.
- [Fowler](https://www.martinfowler.com/bliki/ArchitectureDecisionRecord.html) discusses which decisions merit records, recommends about one page and a reason to revisit the decision, and keeps the original content of a superseded record.
- [adr-tools](https://github.com/npryce/adr-tools) uses numbered filenames and marks the replaced record when a new one supersedes it.
- [adr.github.io](https://adr.github.io/) defines a record as one architectural decision with its rationale and lists the Y-statement form. It confirmed the one-decision rule. The Y-statement was not adopted, because the Context, Decision, and Consequences sections already cover its clauses.
- [Henderson](https://github.com/joelparkerhenderson/architecture-decision-record) describes a good record as specific, timestamped, immutable and backed by a rationale. The guidance on small or temporary decisions informs the worth-keeping item.
- [ThoughtWorks Radar, lightweight ADRs](https://www.thoughtworks.com/radar/techniques/lightweight-architecture-decision-records) rated the technique Adopt and said to keep records in source control. It confirmed the in-repo practice, with no change.
- [Evolu](https://www.evolu.dev/blog/architecture-decision-records-for-coding-agents) discusses decisions worth recording and asks agents to read records before designing and stop when a task conflicts with an accepted record. It warns that parallel sessions can claim the same number.
- [BrainGrid](https://www.braingrid.ai/blog/architecture-decision-records-for-ai-coding-agents) recommends few states so an agent can tell whether a record binds, explicit triggers, and a pointer to the records in the agent's instruction file.
- [AGENTS.md](https://agents.md/) says agents read the nearest instruction file at the repository root. It confirmed that the discovery pointer belongs in the repository's `AGENTS.md`, with no change to the records themselves.
- The [frontmatter schema](../docs/schemas/frontmatter.schema.json) defines the Decision Record fields, the statuses `draft`, `stable`, and `deprecated`, the decision date, and the integer `superseded_by` field for deprecated records.
- [README.md](README.md) gives the disclosure, Decisions entry citation, numbering, decision date, and index rules.
- [TEMPLATE.md](TEMPLATE.md) gives the section and definition rules, the word caps, and the house writing conventions.
- [ADR 0001](https://github.com/tbhb-dev/wiki/blob/22d528cbb73d640ff1291e3a5dd1e9f9ae89a791/adrs/0001-the-changed-paths-and-content-set-the-review-tier.md) sets the decision record review tier and the single delta run for a fix.
- [ADR 0007](https://github.com/tbhb-dev/wiki/blob/22d528cbb73d640ff1291e3a5dd1e9f9ae89a791/adrs/0007-a-review-finding-blocks-when-ordinary-use-breaks.md) sets the ordinary-use test for BLOCK.
- The operator's standing rulings govern self-contained definitions, fidelity to the Decisions entry, and scope boundaries. The disclosure ruling permits private repository names and reserves a stop for credentials or secrets. The house writing ruling sets the sentence, paragraph, list, and wording conventions. Review criteria come from the research summarized here.
