---
type: Decision Record
title: Deliberately evading a check blocks review
description: Deliberate evasion blocks review until the change fixes the underlying issue or uses a reviewed exception.
tags: [reviews, checks, allowlists]
status: stable
decided: "2026-11-11"
---

# Deliberately evading a check blocks review

## Context

A gate is a check required for a change to proceed. It can pass even when the change preserves the condition it was meant to catch. A best-effort check looks for known cases and can miss others. Reviewers need to distinguish deliberate evasion from a gap in such a check.

This blocking rule covers intentional evasion and excludes a missed match on its own.

## Decision

A scanner searches for known patterns. A linter checks code or text against rules. Deliberate evasion of either or any other check is a finding that blocks review. Fix the underlying issue or use the check’s reviewed allowlist, the entries that document permitted exceptions. Redaction placeholders can stand in for information that must be withheld.

## Consequences

A false positive, where the check flags an acceptable change, can get a narrow, documented exception in the check that runs. Removing the allowlist entry restores the default behavior.
