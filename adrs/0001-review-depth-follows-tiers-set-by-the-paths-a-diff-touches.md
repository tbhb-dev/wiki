---
type: Decision Record
title: Review depth follows tiers set by the paths a diff touches
description: A pull request gets no review run, one, or two, by a tier that the paths and content of its diff set, and nothing lowers a tier.
tags: [review, tiers]
status: stable
decided: "2026-11-23"
---

# Review depth follows tiers set by the paths a diff touches

## Context

Every pull request used to need two review runs, whatever it changed. That spent the scarce cross-provider review quota on prose fixes and one-line test changes at the same rate as on credential or deploy code. Asking the maker how risky its own change is would let the riskiest changes pick the lightest review.

## Decision

Tier 0 covers prose-only changes in the wiki, drafts, research and spikes repositories. CI is its only review. Design pull requests, under `designs/` in the wiki repositories, aren't tier 0 and get one review run.

Tier 1 covers most code, tests, refactors and docs in code repositories, and gets one fresh review run.

Tier 2 gets two runs. It covers code that deletes durable data or runs destructive commands, and anything touching credentials, rulesets, permissions, agent harness config, deploys, or the review and merge path. Executable config such as `.mcp.json` counts as harness config at any size. In a development environment, durable data means work that exists nowhere else: uncommitted or unstaged changes, unmerged or local-only branches and worktrees, stashes, records and backups. Resets, checkouts over changes, branch or worktree removal, stash drops and record deletion are tier 2. Deleting regenerable state, such as build caches, Bazel output bases, temporary files and run workspaces, is tier 1.

The tier attaches to code where it changed. A consumer repository that moves a pin to a commit already reviewed and merged in its home repository, with no other change, is tier 1, even when that range touches the review path.

A re-review after a fix covers only the delta, with one run, at any tier. The paths and content a diff touches set its tier, never the maker's judgment. A reviewer may raise a tier, and nothing lowers it.

## Options considered

- Two review runs on every pull request: rejected because it spends review quota where the risk is low.
- A tier chosen by the maker: rejected because the maker is the party least able to judge its own change.

## Consequences

The review-loop design holds the path lists that map a diff to its tier, and the review sessions apply the tiers as an interim rule until that loop exists. Tier 0 merges follow [0002](0002-tier-0-pull-requests-merge-on-green-without-review-or-approval.md), and the provider of each run follows [0003](0003-reviews-come-from-a-provider-other-than-the-builders.md).
