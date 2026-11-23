---
type: Decision Record
title: Review reads each repository's review files from the base branch
description: The review tooling reads a repository's review config, PR template, checklist and schemas from the pull request's base branch, never its head.
tags: [review, config, security]
status: stable
decided: "2026-11-11"
---

# Review reads each repository's review files from the base branch

## Context

Each repository carries files that steer review: `.github/pull_request_review.toml`, the pull request template and checklist, and the draft and semantic schemas under `.github/schemas/`. A baseline checklist applies to every repository. If review read these files from the pull request's head, a change could loosen the rules that review it.

## Decision

Review reads every per-repository review file from the pull request's base branch, never from its head. A pull request that changes the checklist gets the stricter review criteria the review design defines for such changes.

The per-repository review files and their handling belong to the tbhbdev review tooling in tbhb-dev/tools, adapted to the GitHub broker and the rule that agents don't call `gh` directly. They aren't built in ascm for now.

## Consequences

A change to review config takes effect only after it merges, so its own review runs under the old rules. The project-management tooling owns the templates and checks for review content in the shared `tbhbdev.content` package, and the pull request description templates and checks move there later.
