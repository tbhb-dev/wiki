---
type: Decision Record
title: A review finding blocks when ordinary use breaks
description: A finding blocks merge when ordinary use alone produces incorrect results, destroys data or breaks a workflow.
tags: [reviews, findings, debt]
status: stable
decided: "2026-11-23"
---

# A review finding blocks when ordinary use breaks

## Context

A review finding is an issue a reviewer identifies in a proposed change. Reviewers need a common test for whether a finding stops a merge. Ordinary inputs, safe failures and hostile action by a process already using the agent account have different effects on users. The agent account is the system account that runs the agent.

The rule for interference within the agent account applies only when a failure requires such interference, leaving ordinary-use failures outside that exception.

## Decision

Use BLOCK, a finding that stops merge, when ordinary use alone produces incorrect results, destroys data or breaks a workflow. Use SECURITY-DEBT, a security finding that can be fixed later, when the failure requires a process already inside the same agent account to interfere. The security boundary is the account’s separation from processes outside it. Use TECH-DEBT, a technical finding that can be fixed later, when an operation stops safely or leaves state intact.

For best-effort text checks, which can miss some matches, a false refusal of valid text blocks because it breaks ordinary use. A missed match is TECH-DEBT. Save a representative missed case as a test input for a follow-up.

## Consequences

Reviewers classify findings in their own verdicts. The coordinator, who routes work, applies the ordinary-use test to pull requests held in review before waiting on a fix.
