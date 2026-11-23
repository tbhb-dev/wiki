---
type: Decision Record
title: Deliberately evading a check blocks review
description: A change written to escape a scanner, linter or other check blocks until it addresses the finding or carries a reviewed exception.
tags: [reviews, checks, allowlists]
status: stable
decided: "2026-11-11"
---

# Deliberately evading a check blocks review

## Context

A gate can pass even when a change preserves the condition the gate was meant to catch. Reviewers need to distinguish deliberate evasion from a gap in a best-effort check.

## Decision

Treat deliberate evasion of a scanner, linter or other check as a blocking finding. Fix the underlying issue or use an entry in the relevant check’s reviewed allowlist. Redaction placeholders can stand in for information that must be withheld.

## Consequences

A false positive can get a narrow, documented exception in the active check. Removing the allowlist entry restores the default behavior.
