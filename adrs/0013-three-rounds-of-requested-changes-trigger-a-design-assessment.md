---
type: Decision Record
title: Three rounds of requested changes trigger a design assessment
description: Three review cycles that request changes prompt a reassessment of the design and requirements.
tags: [reviews, escalation, design]
status: stable
decided: "2026-11-11"
---

# Three rounds of requested changes trigger a design assessment

## Context

A round counts when a review cycle ends with a request for changes. Successive rounds of fixes can reveal that the design or requirements are wrong. Sending every pull request to a human at a fixed count can miss that underlying problem.

The three-round threshold triggers a design assessment and leaves any handoff to a human to a separate decision.

## Decision

After three rounds of requested changes on a pull request, run a design assessment, an evaluation of whether the design or requirements need revision.

## Consequences

The escalation path, the process for routing unresolved review problems, carries the assessment outcome. A separate decision determines who takes over the work.
