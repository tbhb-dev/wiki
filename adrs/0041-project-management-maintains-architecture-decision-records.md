---
type: Decision Record
title: 0041. Project management maintains architecture decision records
description: Project management maintains the format and commands for decision records, and service management handles optional approval Requests.
tags: [adrs, governance, requests]
status: stable
decided: "2026-11-23"
---

# 0041. Project management maintains architecture decision records

## Context

Architecture decision records state rules that guide designs and planning work. Some decisions need discussion or approval before the rule is ready to publish.

This rule covers decision-record maintenance and its optional approval handoff and leaves the command-line framework and pull request checks with their maintainers on the tooling team.

## Decision

[Project management](0031-work-issues-live-in-the-repositories-responsible-for-them.md) maintains the record format and the rules for replacing an old decision with a new one. It maintains the `tbhbdev adr` commands for listing, showing, creating, superseding, and checking records. Store the records under `adrs/` in the public wiki by default. Use `--private` to select private storage when the details cannot be public.

Use the decision form of a [Request](0034-operator-and-agent-requests-use-github-issues.md) for discussion or approval. Service management runs that issue workflow. After approval, create the record with `tbhbdev adr new --from-request N`, where `N` is the Request's issue number. Close the Request when the record's pull request merges.

## Consequences

Move existing service-management decision files into the wiki's decision record collections. They follow the wiki's pull request process. Decisions that need no Request can start as pull requests for the record.
