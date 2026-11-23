---
type: Decision Record
title: Agents can dismiss reviews and close pull requests with provenance
description: Agents can dismiss GitHub reviews and close pull requests when the action records its provenance.
tags: [reviews, provenance, pull-requests]
status: stable
decided: "2026-11-11"
---

# Agents can dismiss reviews and close pull requests with provenance

## Context

A stale review may need dismissal so it no longer counts. An abandoned pull request may need closure without a merge. For either action, a later reader needs provenance, a record of who acted.

This grant covers dismissing reviews and closing pull requests and leaves other GitHub actions outside its scope.

## Decision

Automated agents can dismiss reviews and close pull requests through the GitHub broker, the service that mediates authorized writes to GitHub. Each dismissal or closure records its provenance.

## Consequences

Dismissal and closure become available through an auditable write path. The provenance record lets a repository maintainer trace who performed either action.
