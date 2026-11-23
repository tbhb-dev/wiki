---
type: Decision Record
title: Approved changes merge without a time-delay wait
description: No merge waits on a timer; once a change has its approval and checks it merges, and safety comes from review, classification, cheap reverts and drift reporting.
tags: [merging, review, safety]
status: stable
decided: "2026-11-11"
---

# Approved changes merge without a time-delay wait

## Context

A design for changes that widen what agents may do proposed a 24-hour wait between approval and merge, so that a bad widening could be caught before it landed. A wait like that holds every such change for a day whether or not anyone looks at it again.

## Decision

No merge waits on a timer. Once a change has its required approval and checks, it merges.

Safety comes from four things instead: review before merge, classification of the change so riskier changes get heavier review, reverts that cost one pull request, and drift reporting that notices a bad state after merge.

## Options considered

- A 24-hour wait after approval for changes that widen permissions: rejected because it stalls work and adds no review.

## Consequences

A control that would rely on a waiting period has to use review, classification, revert or drift reporting instead. Every change lands by pull request, so a revert stays one pull request. Undo: a later decision that adds a wait for a named class of change.
