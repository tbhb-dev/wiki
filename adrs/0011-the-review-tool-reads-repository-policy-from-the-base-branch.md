---
type: Decision Record
title: The review tool reads repository policy from the base branch
description: Repository review settings, templates, checklists and schemas come from the pull request’s base branch.
tags: [reviews, policy, pull-requests]
status: stable
decided: "2026-11-11"
---

# The review tool reads repository policy from the base branch

## Context

A review needs stable criteria while the pull request is under consideration. Otherwise a policy change could affect the review that evaluates it. The base branch is the branch the pull request targets.

The base-branch rule selects repository policy for the current review and excludes policy edits proposed by the pull request under review.

## Decision

The review command in `tbhbdev`, the pull request tooling, reads the repository policy files from the base branch. The settings live in `.github/pull_request_review.toml`. The pull request template specifies the format of a proposal. The checklist specifies the criteria reviewers apply. The draft and semantic schemas in `.github/schemas/` define what a pull request draft and a review result must contain, respectively. The shared baseline checklist, which supplies criteria across repositories, still applies. A pull request that changes the checklist gets the stricter criteria for checklist changes.

## Consequences

The base branch supplies the criteria for reviewing a change to the review files. The new criteria take effect for later pull requests once their base includes the change.
