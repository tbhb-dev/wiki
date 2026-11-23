---
type: Decision Record
title: Renovate pull requests merge automatically once approved and green
description: GitHub automerge merges a Renovate pull request as soon as its required approval and checks are in, with no session's go and no owner merge.
tags: [renovate, dependencies, merging, automerge]
status: stable
decided: "2026-11-11"
---

# Renovate pull requests merge automatically once approved and green

## Context

Renovate opens a steady stream of dependency pull requests (PRs) in every repository. An interim route had each one wait for a session's go and then an owner's merge, so mechanical changes waited on people.

## Decision

Renovate PRs are approved and merged with no session's go in the loop. The shared Renovate preset turns on `automerge` with `platformAutomerge` and keeps its five-day release cooldown, and every repository allows auto-merge. GitHub merges a Renovate PR as soon as its required approval and checks are in.

The approver approves on the CI success event for the PR's head. For update classes that run a command with a token, the approver also confirms that only manifests and lockfiles changed.

A third-party version-and-hash bump needs no review run. A bump of an internal module pin whose range holds hand-written changes, rather than only dependency bumps, gets a review run before approval.

## Options considered

- An owner merges each Renovate PR by hand: rejected because merging needed a person while the change was mechanical.
- A ruleset bypass for the Renovate App: rejected because Renovate PRs pass the same approval and checks as every other PR.

## Consequences

The owner-merge route is gone, and no ruleset's bypass list names the Renovate App. The approval is a step a dedicated approver takes today. In the long term the PR queue worker takes it over.

Renovate rebases only on conflict, so two green bumps can still break main together, and the red-main record covers that case. Undo: set `automerge: false` in the preset, return the approver to needing a go, and turn auto-merge off in the repositories.
