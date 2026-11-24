---
type: Decision Record
title: Renovate pull requests automerge after approval and CI
description: GitHub automerge lands Renovate pull requests once CI and the required approval are complete.
tags: [dependencies, merge, renovate]
status: stable
decided: "2026-11-11"
---

# Renovate pull requests automerge after approval and CI

## Context

Renovate is a bot that opens routine dependency updates. A manual merge handoff delays them after approval and continuous integration (CI), the automated checks on a change. The rule covers merging Renovate's dependency updates and excludes pull requests from other authors.

## Decision

Renovate enables GitHub automerge, which merges eligible updates after required CI and approval. Renovate keeps its five-day cooldown, the minimum release age for an update. The update follows the review tier, or review level, assigned to its changes. Changing only a pin, a fixed commit reference, to an already reviewed and merged commit is tier 1 and gets one review run.

If an update runs a command with Renovate's access token, approval also requires changes to stay within manifests that declare dependencies and lockfiles that record their resolved versions.

## Consequences

The approver handles successful CI notifications until a queue worker, a service that processes queued work, takes over automated approval. GitHub performs the merge once the requirements are met. The release cooldown adds no waiting period after approval. Disabling automerge in the preset shared by repositories restores a separate merge step.

Under the commit conventions rollout, the active ruleset requires both `conclusion` and `pr-title` for GitHub auto-merge and the merge button. The `conclusion` job combines the other required CI results, and the standalone `pr-title` workflow reports the title result. A pull request `edited` event runs only the title workflow. [Required title check](../designs/commit-conventions/D-checker.md#required-title-check)

The title ruleset runs in `evaluate` mode for at least seven days. The administrator account switches it to `active` once those seven days have elapsed and the pre-flip list of open pull requests that would fail is empty. [Commit conventions rollout](../designs/commit-conventions/README.md#rollout)

At the later checker mode flip from `evaluate` to `enforce`, GitHub can still accept an earlier pass from `evaluate` on a head behind the base. Before flipping the mode, the operator lists open pull requests whose titles would fail under `enforce`, fixes the titles or scope configuration, and reruns the check. [Commit conventions rollout](../designs/commit-conventions/README.md#rollout)
