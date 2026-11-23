---
type: Decision Record
title: 0034. Operator and agent requests use GitHub issues
description: Requests make asks durable and use the Handler field, area, and suggested capabilities to route them.
tags: [service-management, requests, routing]
status: stable
decided: "2026-11-23"
---

# 0034. Operator and agent requests use GitHub issues

## Context

A Request is a GitHub issue that records an ask for a decision, action, access, or information so a later worker can pick it up.

Request issues cover routine asks between roles and leave urgent stops and steering of active work to direct messages.

## Decision

The shared Handler field distinguishes Operator, the person with reserved authority, from Agent, an automated worker. On an Agent Request, the area label names the repository whose context the worker loads. Capability labels suggest the skills the work needs. The router matches those skills against the [role registry](https://github.com/tbhb-dev/wiki/blob/main/designs/agent-roles/C-registry.md). It launches a worker of the selected role kind with that repository context.

Choose capability labels from the shared list of skills. The handler or router can revise them. Either can reassign the Request. Routing continues when suggestions are absent or none match. Each run loads its task context without a standing repository assignment.

## Consequences

Keep the operator's current list of open items until the Request workflow goes live. Routine asks between roles move onto issues once the Request commands and agent handling are ready. Existing repository assignments are temporary until workers launch for individual Requests.
