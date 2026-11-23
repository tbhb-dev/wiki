---
type: Decision Record
title: Reviews come from a provider other than the builder's
description: Each review run uses a model from a provider other than the one that built the commits it covers, with a same-provider fallback one tier up.
tags: [review, models, providers]
status: stable
decided: "2026-11-23"
---

# Reviews come from a provider other than the builder's

## Context

A model often misses the mistakes it makes itself, so a review from the builder's own provider adds little. Earlier rules fixed which provider reviewed which builder, and each outage or quota limit needed a new ruling. Pull requests also carry commits from more than one provider, and pin bumps bring in commits that another repository already reviewed.

## Decision

A review run uses a provider other than the builder's. The providers are OpenAI for Codex, Google for Antigravity and Anthropic for Claude, and a diff written by an owner's Claude session counts as Anthropic. No fixed mapping pairs a builder with a reviewer. The reviewer prefers a model of the same tier on a different provider. When only the builder's provider has capacity left, the review uses that provider one tier up, so a Sonnet build gets an Opus reviewer.

When a tier needs two runs, they may share a provider, though two different providers are better when both have quota.

Each commit in the reviewed head needs its tier's number of reviews from providers other than that commit's builder. A review of the whole head counts for every commit in it built by another provider. A head with Codex and Claude commits can meet tier 1 with one Codex review and one Claude review, each covering the other's commits.

The rule covers the commits in a pull request's own head. Upstream commits inside a pinned range got their review in their home repository, so a review of a pin bump may use any provider.

When a provider refuses to review a change, the requester quotes the refusal from the run's log and never rewords the brief to get past it. Another eligible provider takes that review.

## Options considered

- A fixed builder-to-reviewer mapping: rejected because each outage or quota limit needed a new ruling.
- Never allowing the builder's provider: rejected because a provider that runs out of quota would then stall every review.

## Consequences

The models page holds the table of equivalent models across providers and the current routing, which changes more often than this rule. Builders record which run and model wrote a diff, so reviewers can tell each commit's provider.
