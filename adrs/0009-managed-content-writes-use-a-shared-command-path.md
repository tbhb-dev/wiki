---
type: Decision Record
title: Managed content writes use a shared command path
description: Service and project management commands use one content module for templates, validation, and GitHub writes.
tags: [content, service-management, projects]
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

# Managed content writes use a shared command path

## Context

Issue, comment, Project, and pull request writes need a common place to enforce content rules and record what changed.

## Decision

Each kind of service record has a template and a managed command. Group the templates by kind in a directory at the repository root. Commands render from those files. Use the files to generate or check issue forms, and document the procedures in the runbooks.

The service and project management plugins use `tbhbdev.content` for GitHub writes. The module checks the exact text it renders, then writes the issue metadata and relationships and records the operation. These checks include gitleaks, quotation detection, Vale, and validation against the schema for that kind of record. A finding refuses the write and identifies the problem for the author.

## Consequences

As the managed commands become available, the agent hooks and write permissions direct issue, comment, and Project mutations through them. Both plugins use the same write checks, and authors revise refused content before trying again.
