---
type: Decision Record
title: The review worker runs as a restricted Unix service
description: The review and merge worker uses a dedicated Unix service identity and existing GitHub Apps for writes.
tags: [reviews, workers, github]
status: stable
decided: "2026-11-12"
---

# The review worker runs as a restricted Unix service

## Context

The review and merge queue, which holds pending review and merge work, needs to run unattended. Its worker needs access to GitHub. Another GitHub user account would add cost and permissions.

The dedicated service identity covers the unattended worker and leaves interactive agents under their own permissions.

## Decision

Run the worker under a dedicated Unix account on the machine that hosts the service. The worker runs without an agent harness, software that runs an agent. It runs without `sudo`, the command that runs a process with another user’s privileges. The worker obtains credentials from a managed store that controls access to them. It writes to GitHub using tokens issued by existing GitHub Apps, the installed integrations that authorize those writes. It adds no GitHub user account.

## Consequences

The worker can run separately from interactive agents with only its own service permissions. Retiring it includes removing its service identity.
