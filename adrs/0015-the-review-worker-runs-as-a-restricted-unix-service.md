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

The review and merge queue needs to run unattended. The service needs access to GitHub while avoiding the cost and permissions of another GitHub user account.

## Decision

Run the review and merge worker under a dedicated Unix service identity on the service host, without an agent harness or sudo. The worker uses a managed secret store to obtain credentials and writes to GitHub with tokens from the existing apps. It adds no GitHub user account.

## Consequences

The worker can run separately from interactive agents with only its own service permissions. Retiring it includes removing its service identity.
