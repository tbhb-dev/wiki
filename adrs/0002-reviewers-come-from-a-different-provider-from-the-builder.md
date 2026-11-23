---
type: Decision Record
title: Reviewers come from a different provider from the builder
description: Reviewer eligibility is checked against the provider that built each commit.
tags: [reviews, providers, pull-requests]
status: stable
decided: "2026-11-23"
---

# Reviewers come from a different provider from the builder

## Context

A pull request can combine work from several build runs, which are agent runs that produce changes. A provider supplies the model an agent uses to build or review. Treating the branch as one provider’s work can give credit for self-review. It can also discard reviews that cover part of the change.

The provider rule sets which reviews count for each commit and leaves the review count to the [review tier](0001-the-changed-paths-and-content-set-the-review-tier.md).

## Decision

For each commit in the head, the pull request’s latest source commit, count reviews from providers other than the one that built its change. A full-head review covers every commit for which its provider qualifies. If two reviews are required, independent runs from one eligible provider can count. Prefer distinct providers when available.

Prefer a model comparable to the builder’s model using the [model roster](https://github.com/tbhb-dev/wiki/blob/main/models/README.md#matching-tiers). If only the builder’s provider has capacity, use its reviewer model one tier higher. Record any provider refusal verbatim from the run log. Preserve the review instructions for any replacement run.

## Consequences

For a dependency pin update, which changes the selected dependency revision, provider eligibility follows the commits in the repository that uses the dependency. Commits in the dependency itself keep their review in the repository where they landed.
