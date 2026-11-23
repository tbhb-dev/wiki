---
type: Decision Record
title: 0042. Project management defines the format for review verdicts
description: Project management maintains the templates and mechanical checks for review verdicts in the shared content module.
tags: [reviews, content, projects]
status: stable
decided: "2026-11-23"
---

# 0042. Project management defines the format for review verdicts

## Context

An approver reading a review needs to know which revision it covers and how each finding affects the verdict.

## Decision

Project management maintains the templates and checks for review submissions in `tbhbdev.content`. The review records the verdict, the exact commit reviewed, the review tier, and the findings marked `BLOCK`, `TECH-DEBT`, or `SECURITY-DEBT`.

## Consequences

Approvers can read the same fields and finding categories on every review. Templates and checks for pull request descriptions also move into the shared content module.
