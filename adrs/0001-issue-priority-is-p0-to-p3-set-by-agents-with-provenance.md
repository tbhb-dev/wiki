---
type: Decision Record
title: Issue priority is P0 to P3, set by agents with provenance
description: Every issue's priority is one of P0 to P3 in the org Priority field, and agents set it and record why.
tags: [issues, priority, work-tracking]
status: stable
decided: "2026-11-11"
---

# Issue priority is P0 to P3, set by agents with provenance

## Context

Before this rule, the only priority mark on an item was an expedite flag that only the operator could set. Everything else had the same weight, so agents had no shared way to order a backlog, and every reordering waited on the operator. The operator had asked for real priorities.

## Decision

Issue priority takes one of four values, P0 (highest) to P3, in the org Priority field. Agents set and change it. Each setting records its provenance: who set it and the reason, such as the decision or request it rests on. A Problem's priority uses the same field, with its impact and urgency given as the reasoning.

## Options considered

- Keep the expedite flag that only the operator could set: rejected because it ranked nothing below the top and made the operator a bottleneck.

## Consequences

The backlog in the org Project sorts by one field that every repository shares. Because agents set priority, the record of who set it and why is what lets anyone challenge a ranking. This was a reversible call made under the operator's delegation. To undo it, restore the expedite flag and stop agents from setting priority.
