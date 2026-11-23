---
type: Decision Record
title: 0039. Content must pass its CUE schema
description: CUE validates normalized content and metadata, and the shared content module handles a named list of supplemental checks.
tags: [content, cue, validation]
status: stable
decided: "2026-11-22"
---

# 0039. Content must pass its CUE schema

## Context

Validation needs to count the content GitHub renders. It also needs to check metadata and links between records.

These validation rules govern content ready for a GitHub write and leave starter wording to templates.

## Decision

The [content module](0038-managed-content-writes-use-a-shared-command-path.md) converts GitHub Flavored Markdown, GitHub's text format, into blocks such as paragraphs. It measures each section. It adds metadata from the file header or the issue fields. It validates the result against shared content rules and the record type's schema in Configure Unify Execute (CUE). The schema sets headings, limits, metadata rules, and allowed categories.

The module lists supplemental checks for links-only sections, excessive [work notes](0037-service-issue-comments-are-append-only-work-notes.md), approval of a cited [Request](0034-operator-and-agent-requests-use-github-issues.md), and the existence of a required retrospective request for comments. A retrospective request for comments solicits feedback on completed work. These checks report refusals in the same format as schema checks.

## Consequences

Authors get consistent feedback about the rule and the affected section or field. The [coordinator](0036-normal-changes-follow-the-scope-of-delegated-authority.md) can reverse the amendment that allowed supplemental checks by deferring them.
