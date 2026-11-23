---
type: Decision Record
title: 0033. Service workflows run on GitHub issues
description: Service workflows use issues, while reviewed pull requests preserve final reports and configuration records.
tags: [service-management, issues, records]
status: stable
decided: "2026-11-22"
---

# 0033. Service workflows run on GitHub issues

## Context

Service work needs a place to track state and collect evidence. Final reports and configuration records need a reviewable revision history.

This rule covers active service work and the review of final reports and configuration records and leaves [general backlog tracking](0030-work-tracking-uses-github-issues-and-projects.md) to the work-tracking rule.

## Decision

Run each service workflow on a GitHub issue. GitHub Actions, the workflow runner, applies labels. It checks changes in issue state. Use the organization's issue taxonomy, the shared issue types and fields. Use Severity, the shared scale for how serious a failure is. Assess a Problem's Priority from impact and urgency.

An Incident tracks a service disruption. A Problem tracks its cause or the cause of a recurring failure. Submit the final report by pull request after closure or resolution. The configuration management database holds one file per managed component. Change those files by pull request. Both kinds of pull request require continuous integration checks and review by an automated agent.

## Consequences

Each record pull request can merge without rebasing onto the latest base commit. After a push to the default branch, a workflow rebuilds each affected directory's index, its list of records. Contributors use those lists to find records. The workflow maintains them.
