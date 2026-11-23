---
type: Decision Record
title: 0033. Service workflows run on GitHub issues
description: Service workflows use issues, while reviewed pull requests preserve final reports and configuration records.
tags: [service-management, issues, records]
status: stable
decided: "2026-11-22"
generated:
  - role: generated
    harness: codex:0.166.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:15c30f9d922d6894
    at: "2026-11-23T02:46:10Z"
---

# 0033. Service workflows run on GitHub issues

## Context

Service work needs a place to change state and collect evidence. Final reports and configuration records need a reviewable revision history.

## Decision

Run each service workflow on a GitHub issue. GitHub Actions apply labels and check state transitions. Use the organization issue taxonomy and its shared fields. A Problem records the underlying cause of an incident or recurring failure. Use the shared Severity definitions and assess a Problem's Priority from impact and urgency.

When an Incident closes or a Problem resolves, submit its final report by pull request. Configuration management database entries are files for individual configuration items and also change by pull request. These pull requests require CI and agent review.

## Consequences

Each record pull request can merge without a required rebase. After a push to the default branch, a workflow regenerates each affected record directory's index. Contributors use the generated indexes to find records and leave index maintenance to the workflow.
