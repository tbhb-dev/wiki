---
type: Decision Record
title: A review finding blocks when ordinary use breaks
description: Reviewers block failures in ordinary use and record safe failures and threats inside the agent account as debt.
tags: [reviews, findings, debt]
status: stable
decided: "2026-11-23"
---

# A review finding blocks when ordinary use breaks

## Context

Reviewers need a common test for whether a finding stops a merge. Ordinary inputs, safe failures and hostile action by a process that already shares the agent account have different effects on users.

## Decision

Use BLOCK when ordinary use on its own produces incorrect results, destroys data or breaks a workflow. Record a finding as SECURITY-DEBT when it requires interference by a process already inside the same agent account. That account is the security boundary. Record safe failures as TECH-DEBT when the operation stops safely or leaves state intact.

For best-effort text checks, a false refusal of valid text blocks because it breaks ordinary use. A missed match is TECH-DEBT. Save a representative missed case as a fixture for a follow-up.

## Consequences

Reviewers classify findings in their own verdicts. The coordinator applies the ordinary-use test to held pull requests before waiting on a fix.
