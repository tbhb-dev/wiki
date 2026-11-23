---
type: Decision Record
title: Operator and agent requests use GitHub issues
description: Requests make asks durable and use the Handler field, area, and suggested capabilities to route them.
tags: [service-management, requests, routing]
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

# Operator and agent requests use GitHub issues

## Context

A request for a decision, action, access, or information needs a durable record that another run can pick up.

## Decision

The organization Handler field on a Request issue distinguishes Operator from Agent. On an agent Request, the area label identifies the repository context and capability labels suggest the skills the work needs. The router matches those suggestions against the role registry and launches a run of the selected kind with that context.

Choose capability labels from the shared vocabulary. The handler or router can revise them and reassign the Request. Routing continues when suggestions are absent or none match. A launched run loads the context for its task without a standing repository assignment.

## Consequences

The current operator intake remains in use until the Request workflow is live. Routine asks between roles move onto issues when the Request commands and agent handling are ready. Direct messages continue to serve steering and urgent stops. Existing repository assignments are interim during the move to launched runs.
