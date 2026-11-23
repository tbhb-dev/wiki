---
type: Decision Record
title: Reviewers come from a different provider from the builder
description: Review coverage applies per commit, with a higher-tier model on the builder’s provider as the capacity fallback.
tags: [reviews, providers, pull-requests]
status: stable
decided: "2026-11-23"
generated:
  - role: generated
    harness: codex:0.171.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:4f00750704d1bb6e
    at: "2026-11-23T02:44:49Z"
---

# Reviewers come from a different provider from the builder

## Context

A pull request can combine work from several build runs. Treating the entire branch as one provider’s work can either give credit for self-review or discard review that covers part of the change.

## Decision

For each commit in the pull request’s head, count reviews from providers other than the one that built its change. A review of the full head contributes coverage to every commit for which its provider qualifies. The review tier sets the number of qualifying runs. When a pair is needed, two independent runs from one eligible provider can count, with distinct providers preferred when available.

Prefer a model comparable to the builder’s model, using the [model roster](../models/README.md#matching-tiers). If the builder’s provider is the only one with capacity, use a reviewer model one tier higher on that provider. If a provider refuses the review, record the refusal as it appears in the run log and keep the brief intact for any replacement run.

## Consequences

For a dependency pin update, provider eligibility follows the consumer’s own commits. Upstream commits keep their review in the repository where they landed.
