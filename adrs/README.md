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
  - role: generated
    harness: codex:0.166.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:15c30f9d922d6894
    at: "2026-11-23T02:46:10Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Decision records

This public repository holds the decision records that can be stated without anything private. An ADR goes here whenever it can be sanitized without losing its meaning: host names, the tailnet, IP addresses, internal paths, account names and credentials are removed or made generic, and the rule still reads the same. Only a record that would lose its meaning goes in the private wiki repository's `adrs/` folder.

A decision record (ADR) states one decision as a rule in the present tense, with the context that led to it and what follows from it. A record makes sense to a reader of this repository alone.

## Layout

Each record is `adrs/NNNN-<slug>.md`: the next free number, zero-padded to four digits, then a kebab-case slug of the title, such as `0007-squash-merge-every-pull-request.md`. This repository numbers its own records from `0001`. [TEMPLATE.md](TEMPLATE.md) is the house template, and the format and writing rules are in the decision-records design.

## Rules

- A record never names a host, the tailnet, a vault, an internal path, an account or a public IP address. It uses role labels such as "the Linux build host". It may name a private repository.
- A record never cites a Decisions entry or links to a private repository.
- A record never quotes anyone or says who decided. Acceptance is the pull request's approval and merge, and the pull request cites the source of the decision.
- The `decided` field is the date of the decision, not of the record.
- Nobody renames a record after it merges, with one exception: the fix for a duplicate number renames the later file.
- Reviewers use [REVIEW.md](REVIEW.md).

## Records

Add each new record's entry at the end of this list, as `[title](NNNN-slug.md)` and the record's description. When two pull requests add a record at once, the later one rebases and takes the next number.

- [The changed paths and content set the review tier](0001-the-changed-paths-and-content-set-the-review-tier.md) — The files and content a pull request changes set how many reviews it needs.
- [Reviewers come from a different provider from the builder](0002-reviewers-come-from-a-different-provider-from-the-builder.md) — Reviewer eligibility is checked against the provider that built each commit.
- [The role that commissions a build can launch its reviews](0003-the-role-that-commissions-a-build-can-launch-its-reviews.md) — The role that starts a build can also launch its reviews.
- [Review runs publish verdicts on the pull request](0004-review-runs-publish-verdicts-on-the-pull-request.md) — Reviewers publish their own results for the specific commit they examined.
- [Check results start reviews for the matching head](0005-check-results-start-reviews-for-the-matching-head.md) — Automated check results start queued reviews or sizing reviews for the commits they checked.
- [Pull requests in one repository can be reviewed in parallel](0006-pull-requests-in-one-repository-can-be-reviewed-in-parallel.md) — Pull requests can enter review as capacity permits, including when one builds on another.
- [A review finding blocks when ordinary use breaks](0007-a-review-finding-blocks-when-ordinary-use-breaks.md) — A finding blocks merge when ordinary use alone produces incorrect results, destroys data or breaks a workflow.
- [Deliberately evading a check blocks review](0008-deliberately-evading-a-check-blocks-review.md) — Deliberate evasion blocks review until the change fixes the underlying issue or uses a reviewed exception.
- [The reviews app posts approvals for an independent approver](0009-the-reviews-app-posts-approvals-for-an-independent-approver.md) — The reviews app posts approval after an independent approver verifies the proposed revision and its review evidence.
- [A rebuilt pull request can reuse approval for an identical tree](0010-a-rebuilt-pull-request-can-reuse-approval-for-an-identical-tree.md) — A rebuilt pull request still needs a full-diff review at its exact head when its Git tree matches the approved tree.
- [The review tool reads repository policy from the base branch](0011-the-review-tool-reads-repository-policy-from-the-base-branch.md) — Repository review settings, templates, checklists and schemas come from the pull request’s base branch.
- [Agents can dismiss reviews and close pull requests with provenance](0012-agents-can-dismiss-reviews-and-close-pull-requests-with-provenance.md) — Agents can dismiss GitHub reviews and close pull requests when the action records its provenance.
- [Three rounds of requested changes trigger a design assessment](0013-three-rounds-of-requested-changes-trigger-a-design-assessment.md) — Three review cycles that request changes prompt a reassessment of the design and requirements.
- [Merge commits record the merger](0014-merge-commits-record-the-merger.md) — A Merged-by line in the commit message identifies who merged the pull request.
- [The review worker runs as a restricted Unix service](0015-the-review-worker-runs-as-a-restricted-unix-service.md) — The review and merge worker uses a dedicated Unix service identity and existing GitHub Apps for writes.
- [The review worker can check GitHub every 15 minutes](0016-the-review-worker-can-check-github-every-15-minutes.md) — The review and merge worker can use a periodic GitHub read to recover missed events.
- [Automated review can approve delegated permission increases](0017-automated-review-can-approve-delegated-permission-increases.md) — Automated review may approve permission increases for narrow standing allow rules under deploy policy.
- [Agents use tbhbdev for GitHub operations](0018-agents-use-tbhbdev-for-github-operations.md). Agents use dedicated tbhbdev commands for GitHub reads and writes, and shared checks validate content before posting.
- [Agents close pull requests with a reason](0019-agents-close-pull-requests-with-a-reason.md). Agents close pull requests through tbhbdev, record the reason and keep the branch available for reopening.
- [Pull requests land through merge commits](0020-pull-requests-land-through-merge-commits.md). GitHub lands pull requests with merge commits so dependent pull requests keep their commit history.
- [Authors may update and curate pull request branches](0021-authors-may-update-and-curate-pull-request-branches.md). Authors may merge in or rebase onto the default branch and use a lease for any force-push to a pull request branch.
- [Approved changes merge without a fixed delay](0022-approved-changes-merge-without-a-fixed-delay.md). An eligible pull request can merge as soon as its required checks and approvals are complete.
- [Repositories fix a failing default branch first](0023-repositories-fix-a-failing-default-branch-first.md). Pull request branches can merge without rebasing, and a failing default branch takes priority over other work.
- [Renovate pull requests automerge after approval and CI](0024-renovate-pull-requests-automerge-after-approval-and-ci.md). GitHub automerge lands Renovate pull requests once CI and the required approval are complete.
- [Clone main worktrees stay clean and on main](0025-clone-main-worktrees-stay-clean-and-on-main.md). The launcher, command guard, Git hook and scheduled audit keep each clone's main worktree clean and on its default branch.
- [Pull request cycle time starts at the earliest recorded time](0026-pull-request-cycle-time-starts-at-the-earliest-recorded-time.md). Pull request cycle time runs from the earliest recorded agent start or commit time to merge or unmerged close.
- [tbhb-dev has one definition of done](0027-tbhb-dev-has-one-definition-of-done.md). The rules repository owns a shared definition of done and issue types define the shape of their acceptance criteria.
- [CI status records check results](0028-ci-status-records-check-results.md). Pull request bodies, issue bodies and acceptance criteria leave CI results and CI-enforced items to the checks themselves.
- [Wording-only follow-ups join the next needed pull request](0029-wording-only-follow-ups-join-the-next-needed-pull-request.md). A follow-up that changes wording alone joins the next needed pull request for the same work.
- [0030. Work tracking uses GitHub Issues and Projects](0030-work-tracking-uses-github-issues-and-projects.md). GitHub Issues hold backlog items from intake onward, and organization Projects show their priority and status.
- [0031. Work issues live in the repositories responsible for them](0031-work-issues-live-in-the-repositories-responsible-for-them.md). Responsible repositories hold local work and Features (units of user-visible work), while a central planning repository holds Initiatives and Epics.
- [0032. Issues use the public repository when their text can be public](0032-issues-use-the-public-repository-when-their-text-can-be-public.md). For a public and private repository pair, issue text determines which repository receives the issue.
- [0033. Service workflows run on GitHub issues](0033-service-workflows-run-on-github-issues.md). Service workflows use issues, while reviewed pull requests preserve final reports and configuration records.
- [0034. Operator and agent requests use GitHub issues](0034-operator-and-agent-requests-use-github-issues.md). Requests make asks durable and use the Handler field, area, and suggested capabilities to route them.
- [0035. Change issues use three approval classes](0035-change-issues-use-three-approval-classes.md). Change issues come from change Requests and follow Standard, Normal, or Emergency approval rules.
- [0036. Normal Changes follow the scope of delegated authority](0036-normal-changes-follow-the-scope-of-delegated-authority.md). The responsible role, the coordinator, or the operator handles a Normal Change according to its scope and the authority it requires.
- [0037. Service issue comments are append-only work notes](0037-service-issue-comments-are-append-only-work-notes.md). Structured work notes record updates, evidence, and state transitions on service issues.
- [0038. Managed content writes use a shared command path](0038-managed-content-writes-use-a-shared-command-path.md). Service and project management commands use one content module for templates, validation, and GitHub writes.
- [0039. Content must pass its CUE schema](0039-content-must-pass-its-cue-schema.md). CUE validates normalized content and metadata, and the shared content module handles a named list of supplemental checks.
- [0040. Service and project management designs live in the public wiki](0040-service-and-project-management-designs-live-in-the-public-wiki.md). The public wiki is the home for the service-management and project-management designs.
- [0041. Project management maintains architecture decision records](0041-project-management-maintains-architecture-decision-records.md). Project management maintains the format and commands for decision records, and service management handles optional approval Requests.
- [0042. Project management defines the format for review verdicts](0042-project-management-defines-the-format-for-review-verdicts.md). Project management maintains the templates and mechanical checks for review verdicts in the shared content module.
- [0043. A workflow can approve pin-only Renovate pull requests](0043-a-workflow-can-approve-pin-only-renovate-pull-requests.md). A reusable workflow and a dedicated GitHub App approve Renovate pull requests that only move pins and pass the required checks.
