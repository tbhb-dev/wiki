---
type: Decision Record
title: The reviews app posts approvals for an independent approver
description: The reviews app posts approval for a verified green head after an independent approver checks the review evidence.
tags: [approvals, reviews, github]
status: stable
decided: "2026-11-23"
---

# The reviews app posts approvals for an independent approver

## Context

A reviewer’s verdict and a GitHub approval serve different roles. Approval needs its own identity check so that the maker or opener cannot approve the same work.

## Decision

The GitHub reviews app posts an approval through `review create`. The independent approver reads the full diff and the required review evidence for the exact head. Approval requires a successful `conclusion` check and verified status on every commit GitHub lists. The service uses the launcher’s host-side run record to reject an approval from the same root identity that opened the pull request.

An approver role can run on demand. The same independence and evidence checks apply to that run.

## Consequences

Until the app can post approvals, the designated approver uses the existing approval command with the exact head. The coordination loop can act on a reviewer’s approval only after leaving shadow mode and confirming that the reviewer read the full diff and the head passed the conclusion check.
