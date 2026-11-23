---
type: Decision Record
title: 0039. Content must pass its CUE schema
description: CUE validates normalized content and metadata, and the shared content module handles a named list of supplemental checks.
tags: [content, cue, validation]
status: stable
decided: "2026-11-22"
generated:
  - role: generated
    harness: codex:0.166.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:15c30f9d922d6894
    at: "2026-11-23T02:46:10Z"
---

# 0039. Content must pass its CUE schema

## Context

A starter template helps an author write. Validation also needs to count the content GitHub renders and check metadata and relationships.

## Decision

Templates provide starting text. The content module normalizes GitHub Flavored Markdown into blocks and section metrics, adds frontmatter or issue metadata, and validates the result against the content type's CUE variant and shared policy. The schema defines the applicable headings, limits, metadata, and taxonomy.

The module defines checks beyond CUE in an explicit list. They cover links-only sections, flooding an issue with work notes, approval of a cited Request, and the existence of a required retrospective request for comments. These checks report refusals in the same format as schema checks.

## Consequences

Authors get consistent feedback about the rule and the affected section or field. Coordination can reverse the amendment by deferring the supplemental checks.
