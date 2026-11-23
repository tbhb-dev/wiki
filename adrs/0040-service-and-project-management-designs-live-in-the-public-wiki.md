---
type: Decision Record
title: 0040. Service and project management designs live in the public wiki
description: The public wiki is the home for the service-management and project-management designs.
tags: [designs, service-management, projects]
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

# 0040. Service and project management designs live in the public wiki

## Context

The service-management and project-management designs explain how work moves across repositories. Readers need one place to find them and follow their changes.

## Decision

Maintain the [service-management design](../designs/itsm-records/README.md) and [project-management design](../designs/pm/README.md) in the public wiki. Identify private components by their role. Apply the public-content checks before publication.

## Consequences

Both designs follow the wiki's pull request process. The public descriptions explain the workflows for readers outside the organization.
