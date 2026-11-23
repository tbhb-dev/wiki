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

## Decision

File each bug or task in the repository responsible for it. Use the central planning repository for Initiatives, Epics, and Stories. Individual issues link to their planning parents as sub-issues, including across repositories. Organization Projects bring the issues together.

The repository for project management supplies shared tools and workflows. Its issue queue tracks work on those tools and workflows.

## Consequences

An issue stays with the code or service it affects while its parent tracks the wider outcome. One Project can show work across repositories without duplicating the issues.
