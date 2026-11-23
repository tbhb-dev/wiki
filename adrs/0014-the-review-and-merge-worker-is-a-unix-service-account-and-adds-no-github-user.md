---
type: Decision Record
title: The review and merge worker is a Unix service account and adds no GitHub user
description: The review and merge worker runs as a Unix service account with no agent harness and no sudo, and writes to GitHub only through the existing Apps.
tags: [review, merge-queue, accounts]
status: stable
decided: "2026-11-12"
---

# The review and merge worker is a Unix service account and adds no GitHub user

## Context

The review and merge worker is a service that tracks pull requests through review and merge. It needs an identity on its host and a way to write to GitHub. A GitHub user account for it would cost a paid seat in the organization.

## Decision

The worker runs as a Unix service account on the service host. It has no agent harness and no sudo, and it holds a token for the secrets manager. Its GitHub writes go through the existing GitHub Apps' tokens.

The build adds no GitHub user account.

## Consequences

The worker can't run model sessions or gain root on its host, so a fault in it stays inside one service account. Its GitHub permissions are whatever the existing Apps grant, and widening them is an App change. Undoing this means removing the service account.
