---
type: Decision Record
title: 0030. Work tracking uses GitHub Issues and Projects
description: GitHub Issues hold backlog items from intake onward, and organization Projects show their priority and status.
tags: [issues, projects, tracking]
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

# 0030. Work tracking uses GitHub Issues and Projects

## Context

A backlog kept in separate documents makes people reconcile progress and wait for a build slot before giving new work a durable record.

## Decision

A GitHub issue can enter the backlog at any point, including before anyone can start the work. The organization Project shows its priority and status. Use the shared Priority field and record the reason for a priority change in a work note. Refine the issue when someone picks it up, and give each fact one authoritative home.

## Consequences

Routine progress is visible on the issue or Project. Keep using the trackers until the initial templates are ready for import. Use the existing security debt list until the affected private repositories take over those entries.
