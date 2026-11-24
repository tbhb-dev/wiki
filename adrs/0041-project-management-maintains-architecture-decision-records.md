---
type: Decision Record
title: 0041. Project management maintains architecture decision records
description: Project management maintains the decision record format and commands, and service management owns the Decision workflow that can start a record.
tags: [adrs, governance, decisions]
status: stable
decided: "2026-11-24"
---

# 0041. Project management maintains architecture decision records

## Context

Architecture decision records state rules that guide designs and planning work. Some decisions need discussion or approval before the rule is ready to publish.

A [Request](0034-operator-and-agent-requests-use-github-issues.md) asks for work. A Decision is a separate GitHub issue type that asks for an answer. The decider is the role responsible for answering the Decision.

This rule covers decision record maintenance and the handoff from Decision issues. The tooling team continues to maintain the command-line framework and pull request checks.

## Decision

[Project management](0031-work-issues-live-in-the-repositories-responsible-for-them.md) maintains the record format, rules for replacing records, and `tbhbdev adr` commands for listing, showing, creating, superseding, and checking records. Store records under `adrs/` in the public wiki by default. Use `--private` to select private storage when the details cannot be public.

[Service management](0033-service-workflows-run-on-github-issues.md) owns the Decision type, template, and workflow. To propose a record, the Decision's author applies the `adr` label. The decider may change the label when answering.

Without `adr`, record the answer and close the issue. With `adr`, draft the record using `tbhbdev adr new --from-decision N`, where `N` is the Decision's issue number. Close the issue when the record's pull request merges. A Decision that supersedes a record names the replaced record's number.

## Consequences

Move existing service-management decision files into the wiki's decision record collections. They follow the wiki's pull request process. A record can also start as a pull request without a Decision.
