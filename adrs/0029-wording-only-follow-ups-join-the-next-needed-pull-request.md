---
type: Decision Record
title: Wording-only follow-ups join the next needed pull request
description: A follow-up that changes wording alone joins the next needed pull request for the same work.
tags: [documentation, pull-requests, review]
status: stable
decided: "2026-11-22"
---

# Wording-only follow-ups join the next needed pull request

## Context

Opening a separate pull request for each small wording correction repeats submission and review work for the same change. The rule covers later edits to phrasing for the same work and excludes changes to requirements or behavior.

## Decision

A wording-only follow-up, an edit limited to phrasing, goes into the next pull request that the same work already requires.

## Consequences

A deferred wording correction waits for that pull request. Requiring a separate submission again reverses the rule.
