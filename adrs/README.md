---
type: Readme
title: Decision records
description: Which decisions this repository records, the record layout, and the index of records.
tags: [adrs, decisions, index]
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

- [Agents use tbhbdev for GitHub operations](0001-agents-use-tbhbdev-for-github-operations.md). Agents use dedicated tbhbdev commands for GitHub reads and writes, and shared checks validate content before posting.
- [Agents close pull requests with a reason](0002-agents-close-pull-requests-with-a-reason.md). Agents close pull requests through tbhbdev, record the reason and keep the branch available for reopening.
- [Pull requests land through merge commits](0003-pull-requests-land-through-merge-commits.md). GitHub lands pull requests with merge commits so dependent pull requests keep their commit history.
- [Authors may update and curate pull request branches](0004-authors-may-update-and-curate-pull-request-branches.md). Authors may merge in or rebase onto the default branch and use a lease for any force-push to a pull request branch.
- [Approved changes merge without a fixed delay](0005-approved-changes-merge-without-a-fixed-delay.md). An eligible pull request can merge as soon as its required checks and approvals are complete.
- [Repositories fix a failing default branch first](0006-repositories-fix-a-failing-default-branch-first.md). Pull request branches can merge without rebasing, and a failing default branch takes priority over other work.
- [Renovate pull requests automerge after approval and CI](0007-renovate-pull-requests-automerge-after-approval-and-ci.md). GitHub automerge lands Renovate pull requests once CI and the required approval are complete.
- [Clone main worktrees stay clean and on main](0008-clone-main-worktrees-stay-clean-and-on-main.md). The launcher, command guard, Git hook and scheduled audit keep each clone's main worktree clean and on its default branch.
- [Pull request cycle time starts with the oldest commit trailer](0009-pull-request-cycle-time-starts-with-the-oldest-commit-trailer.md). Pull request cycle time runs from the earliest start recorded in its commit identity trailers to its merge or unmerged close.
- [tbhb-dev has one definition of done](0010-tbhb-dev-has-one-definition-of-done.md). The rules repository owns a shared definition of done and issue types define the shape of their acceptance criteria.
- [CI status records check results](0011-ci-status-records-check-results.md). Pull request bodies, issue bodies and acceptance criteria leave CI results and CI-enforced items to the checks themselves.
- [Wording-only follow-ups join the next needed pull request](0012-wording-only-follow-ups-join-the-next-needed-pull-request.md). A follow-up that changes wording alone joins the next needed pull request for the same work.
