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

A review needs stable criteria while the pull request is under consideration. Otherwise a change to the policy could affect the review that decides whether the policy should land.

## Decision

The `tbhbdev` review tool reads `.github/pull_request_review.toml`, the pull request template and checklist, and the draft and semantic schemas in `.github/schemas/` from the pull request’s base branch. The shared baseline checklist still applies. A pull request that changes the checklist gets the stricter review criteria for checklist changes.

## Consequences

The base branch supplies the criteria for reviewing a change to the review files. The new criteria take effect for later pull requests once their base includes the change.
