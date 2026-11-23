---
type: Decision Record
title: Code written to dodge a check is a blocking finding
description: Code shaped so a scanner, lint or check won't match is always a blocking review finding, fixed at the cause or through a reviewed allowlist entry.
tags: [review, findings, checks]
status: stable
decided: "2026-11-11"
---

# Code written to dodge a check is a blocking finding

## Context

Scanners, lints and checks match text. A change can pass one by building a string from parts, or by reshaping a literal so duplicate-code detection no longer sees it. The check then passes while the thing it guards against is still there, and nobody reviews the exemption.

## Decision

Code written so a scanner, lint or check won't match is a blocking review finding, never a note. That covers strings built from parts, literals reshaped to slip past duplicate-code detection, and similar evasions.

The fix either addresses what the check is about, or adds a reviewed allowlist entry with a comment that gives the reason. Placeholder text that redacts a value is fine.

## Consequences

When a check refuses a legitimate literal, the author adds that exact literal to the check's own exemption list in a reviewed change and keeps the literal whole in source and tests. Reviewers treat a split or disguised literal as a BLOCK even when the code otherwise works, so [0007](0007-a-finding-blocks-only-when-normal-use-breaks.md)'s normal-use test doesn't lower it.
