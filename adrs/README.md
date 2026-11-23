---
type: Readme
title: Decision records
description: Which decisions this repository records, the record layout, and the index of records.
tags: [adrs, decisions, index]
status: draft
generated:
  - role: generated
    harness: codex:0.171.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:4f00750704d1bb6e
    at: "2026-11-23T02:44:49Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Decision records

This public repository holds the decision records that can be stated without anything private. An ADR goes here whenever it can be sanitized without losing its meaning: host names, the tailnet, IP addresses, internal paths, account names and credentials are removed or made generic, and the rule still reads the same. Only a record that would lose its meaning goes in the private wiki repository's `adrs/` folder.

A decision record (ADR) states one decision as a rule in the present tense, with the context that led to it and what follows from it. A record makes sense to a reader of this repository alone.

## Layout

Each record is `adrs/NNNN-<slug>.md`: the next free number, zero-padded to four digits, then a kebab-case slug of the title, such as `0007-squash-merge-every-pull-request.md`. This repository numbers its own records from `0001`. [TEMPLATE.md](TEMPLATE.md) is the house template, and the format and writing rules are in the decision-records design.

## Rules

- A record never names a host, the tailnet, a vault, an internal path, an account, a private repository or a public IP address. It uses role labels such as "the Linux build host".
- A record never cites a Decisions note or links to a private repository.
- A record never quotes anyone or says who decided. Acceptance is the pull request's approval and merge, and the pull request cites the source of the decision.
- The `decided` field is the date of the decision, not of the record.
- Nobody renames a record after it merges, with one exception: the fix for a duplicate number renames the later file.

## Records

Add each new record's entry at the end of this list, as `[title](NNNN-slug.md)` and the record's description. When two pull requests add a record at once, the later one rebases and takes the next number.

- [The changed paths and content set the review tier](0001-the-changed-paths-and-content-set-the-review-tier.md) — Review tiers set how many runs a pull request needs, and a fix gets one review of its delta.
- [Reviewers come from a different provider from the builder](0002-reviewers-come-from-a-different-provider-from-the-builder.md) — Review coverage applies per commit, with a higher-tier model on the builder’s provider as the capacity fallback.
- [The role that commissions a build can launch its reviews](0003-the-role-that-commissions-a-build-can-launch-its-reviews.md) — The role that commissions a build can start its review runs and leaves the verdicts to those runs.
- [Review runs publish verdicts on the pull request](0004-review-runs-publish-verdicts-on-the-pull-request.md) — Reviewers publish their own results for the exact head, and approvers and mergers verify them at the source.
- [Check results start reviews for the matching head](0005-check-results-start-reviews-for-the-matching-head.md) — A successful conclusion check starts a queued review, and an isolated size failure starts a sizing review.
- [Pull requests in one repository can be reviewed in parallel](0006-pull-requests-in-one-repository-can-be-reviewed-in-parallel.md) — Each qualifying head enters review as capacity permits, including heads in a stack of pull requests.
- [A review finding blocks when ordinary use breaks](0007-a-review-finding-blocks-when-ordinary-use-breaks.md) — Reviewers block failures in ordinary use and record safe failures and threats inside the agent account as debt.
- [Deliberately evading a check blocks review](0008-deliberately-evading-a-check-blocks-review.md) — A change written to escape a scanner, linter or other check blocks until it addresses the finding or carries a reviewed exception.
- [The reviews app posts approvals for an independent approver](0009-the-reviews-app-posts-approvals-for-an-independent-approver.md) — The reviews app posts approval for a verified green head after an independent approver checks the review evidence.
- [A rebuilt pull request can reuse approval for an identical tree](0010-a-rebuilt-pull-request-can-reuse-approval-for-an-identical-tree.md) — The review worker can approve a replacement without a new model read when the rebuilt Git tree matches the approved tree.
- [The review tool reads repository policy from the base branch](0011-the-review-tool-reads-repository-policy-from-the-base-branch.md) — Repository review settings, templates, checklists and schemas come from the pull request’s base branch.
- [Agents can dismiss reviews and close pull requests with provenance](0012-agents-can-dismiss-reviews-and-close-pull-requests-with-provenance.md) — Agents can dismiss GitHub reviews and close pull requests when the action records its provenance.
- [Three rounds of requested changes trigger a design assessment](0013-three-rounds-of-requested-changes-trigger-a-design-assessment.md) — Repeated requests for changes prompt an assessment of the design and requirements without an automatic handoff to a human.
- [Merge commits record the merger](0014-merge-commits-record-the-merger.md) — The merge commit identifies the actor responsible for merging the pull request in a Merged-by trailer.
- [The review worker runs as a restricted Unix service](0015-the-review-worker-runs-as-a-restricted-unix-service.md) — The review and merge worker uses a dedicated Unix service identity and existing GitHub Apps for writes.
- [The review worker can check GitHub every 15 minutes](0016-the-review-worker-can-check-github-every-15-minutes.md) — The review and merge worker can use a periodic GitHub read to recover missed events.
- [Automated review can approve delegated permission increases](0017-automated-review-can-approve-delegated-permission-increases.md) — Deploy policy allows automated approval for class R permission increases and reserves class D increases for the operator.
