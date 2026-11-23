---
type: Decision Record
title: 0031. Work issues live in the repositories responsible for them
description: Repository issues hold local work, and a central planning repository holds Initiatives, Epics, and Stories.
tags: [issues, projects, planning]
status: stable
decided: "2026-11-22"
---

# 0031. Work issues live in the repositories responsible for them

## Context

People need to find an individual change with the code it affects and see how that change contributes to work that spans repositories.

This rule assigns repositories to work issues and planning items and leaves the [choice between public and private counterparts](0032-issues-use-the-public-repository-when-their-text-can-be-public.md) to the publication rules.

## Decision

File each bug or task in the repository responsible for it. Keep Initiatives, which track broad goals, in the shared planning repository. Keep Epics, which group related work, there as well. Keep Stories, which describe user-facing outcomes, in the same repository. Link each individual issue to its larger planning item as a GitHub sub-issue, including across repositories. [Organization Projects](0030-work-tracking-uses-github-issues-and-projects.md) bring the issues together.

The repository for project management supplies shared tools and workflows. Its issue queue tracks work on those tools and workflows.

## Consequences

An issue stays with the code or service it affects while its parent tracks the wider outcome. One Project can show work across repositories without duplicating the issues.
