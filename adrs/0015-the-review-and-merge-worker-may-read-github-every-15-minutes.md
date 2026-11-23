---
type: Decision Record
title: The review and merge worker may read GitHub every 15 minutes
description: The review and merge worker reads GitHub every 15 minutes to catch lost events, as a sanctioned exception to the rule against polling.
tags: [review, merge-queue, events]
status: stable
decided: "2026-11-12"
---

# The review and merge worker may read GitHub every 15 minutes

## Context

Agents and services react to GitHub events rather than polling, and a standing rule forbids polling. GitHub can still drop a webhook delivery. A worker that only listens for events would leave a pull request stuck whenever its event goes missing.

## Decision

The review and merge worker may read GitHub every 15 minutes to catch lost events. This read is a sanctioned exception to the rule against polling, and the exception covers this worker only.

## Consequences

A lost event delays a pull request by at most about 15 minutes instead of stranding it. The read draws on the worker's API rate limit at a fixed, low rate. Other services still subscribe to events and don't poll. Undoing this means reverting the exception.
