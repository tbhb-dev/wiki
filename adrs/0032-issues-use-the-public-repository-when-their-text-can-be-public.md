---
type: Decision Record
title: 0032. Issues use the public repository when their text can be public
description: For a public and private repository pair, issue text determines which repository receives the issue.
tags: [issues, privacy, routing]
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

# 0032. Issues use the public repository when their text can be public

## Context

A project with public and private repositories needs a default issue destination and a way to keep sensitive operational details private.

## Decision

Open the issue in the public repository when its text can be public. When the text needs details barred from public content, use the private counterpart. The project management tooling looks up the counterpart at runtime using the organization's naming convention and checks the exact rendered text before a public write. A private-detail finding refuses the public destination and offers the private one. A secret finding refuses the write.

Use GitHub's issue transfer when moving an existing issue between the two repositories, with the same scan before the move. Preserve the author's text through routing.

## Consequences

Public work remains visible by default. A scan result explains why the public destination is unavailable. Each item has one issue across the pair.
