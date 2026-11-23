---
type: Decision Record
title: Check results start reviews for the matching head
description: Automated check results start queued reviews or sizing reviews for the commits they checked.
tags: [reviews, checks, events]
status: stable
decided: "2026-11-22"
---

# Check results start reviews for the matching head

## Context

A notice from the owner responsible for the pull request may arrive late. It may describe an earlier head, the latest commit on the pull request's source branch at the time of the check. A pull request whose only failure concerns the size of its change can wait indefinitely for a passing check.

These triggers start queued reviews or sizing reviews for the checked head and leave queued reviews for other heads waiting.

## Decision

Queue reviews against the head to be reviewed. A GitHub `check_run` event reports an automated check result for a commit. A successful result for `conclusion`, the check that summarizes the required jobs, releases a queued review if its commit matches. When the size check alone fails, start the sizing review as soon as the result is available. The sizing review judges whether the proposed change is an acceptable size.

## Consequences

Owners can rely on check events to start reviews. Reverting the event trigger restores owner notifications, and reverting the sizing trigger restores an explicit sizing request.
