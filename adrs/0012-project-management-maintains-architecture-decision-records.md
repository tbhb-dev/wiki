---
type: Decision Record
title: Project management maintains architecture decision records
description: Project management maintains the format and commands for decision records, and service management handles optional approval Requests.
tags: [adrs, governance, requests]
status: stable
decided: "2026-11-23"
generated:
  - role: generated
    harness: codex:0.166.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:15c30f9d922d6894
    at: "2026-11-23T02:46:10Z"
---

# Project management maintains architecture decision records

## Context

Architecture decisions guide designs and planning work. Some decisions also need a question and approval workflow before the rule is ready to publish.

## Decision

Project management maintains the architecture decision record format, the rules for superseding records, and the `tbhbdev adr` commands for listing, showing, creating, superseding, and checking records. The tooling team provides the command-line framework and pull request checks. Store the records under `adrs/` in the public wiki by default. Use `--private` for decisions that require details unsuitable for publication.

Use the decision Request variant when a decision needs discussion or approval. Service management runs that issue workflow. Approval hands off to `tbhbdev adr new --from-request N`, and the Request closes when the record's pull request merges.

## Consequences

Existing service-management decision files move into the wiki collections, where they follow the wiki's pull request process. Decisions that don't need a Request can start as pull requests for the record.
