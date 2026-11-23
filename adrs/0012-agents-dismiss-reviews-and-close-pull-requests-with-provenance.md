---
type: Decision Record
title: Agents dismiss reviews and close pull requests, with provenance
description: Agents have full control over dismissing reviews and closing pull requests as long as each act records its provenance, and merges carry a Merged-by trailer.
tags: [review, provenance, github-broker]
status: stable
decided: "2026-11-11"
---

# Agents dismiss reviews and close pull requests, with provenance

## Context

Agents reach GitHub through a broker that decides which write verbs they may use. Dismissing a stale review and closing a superseded pull request are routine steps in agent work, and holding them for a human would stall it. Both acts also remove a signal from view, so a later reader needs to know who did it and why.

## Decision

Agents have full control over dismissing reviews and closing pull requests through the broker. Each dismissal or close records its provenance: which agent acted, on what authority and why.

A Merged-by trailer joins the fixed set of commit trailer keys, so a merge records who merged it.

## Consequences

The broker design carries the provenance requirement into its dismissal and close verbs. The trailer checks accept Merged-by. Undoing either part means narrowing the broker's verbs or removing the trailer key from the fixed set.
