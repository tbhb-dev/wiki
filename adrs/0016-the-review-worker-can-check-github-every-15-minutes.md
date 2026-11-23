---
type: Decision Record
title: The review worker can check GitHub every 15 minutes
description: The review and merge worker can use a periodic GitHub read to recover missed events.
tags: [reviews, workers, events]
status: stable
decided: "2026-11-12"
---

# The review worker can check GitHub every 15 minutes

## Context

A missed event can leave a pull request waiting in the review or merge queue after its state changes on GitHub.

## Decision

The review and merge worker may read GitHub every 15 minutes to reconcile state after lost events. This is a standing exception to the general limit on polling.

## Consequences

Events remain the normal way to advance the queue. Removing the exception returns the worker to event delivery alone.
