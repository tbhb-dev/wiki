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

The review and merge worker is the service that processes pending reviews and merges. A missed event, a GitHub notice of a state change, can leave a pull request waiting after its state changes.

## Decision

The worker may read GitHub every 15 minutes to bring the queue up to date after missed events. This is a standing exception to the general limit on polling, or checking GitHub on a timer. The exception covers recovery from missed events and leaves event delivery as the normal way to advance the queue.

## Consequences

Removing the exception returns the worker to event delivery alone.
