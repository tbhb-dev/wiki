---
type: Decision Record
title: pm owns the templates and checks for review-run output
description: What a review run posts on a pull request follows templates and mechanical checks that pm owns in the content kernel.
tags: [reviews, content, pm, templates]
status: stable
decided: "2026-11-23"
---

# pm owns the templates and checks for review-run output

## Context

Review runs post their own verdicts on pull requests, and approvers read those verdicts directly. A verdict that leaves out the head it judged, its tier or the class of a finding can't be acted on safely. Nothing yet fixed the shape of that output or checked it mechanically.

## Decision

The tbhb-dev/pm repository owns the templates and mechanical checks for what review runs post: the verdict, the exact head, the review tier, and findings each marked BLOCK, TECH-DEBT or SECURITY-DEBT. They're built in the `tbhbdev.content` package, like every other content type. Later, the existing pull request description templates and checks move into the same package.

## Consequences

Review output gets the same refusal-on-finding treatment as issues and records, so a verdict missing its head or tier is refused before it's posted. The pull request description checks end up in one place with the rest of the content checks.
