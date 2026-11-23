---
type: Decision Record
title: Templates give starting text, and CUE schemas decide validity
description: Each content type has a template file for its starting text and a CUE schema that decides whether text is valid, with a short named list of kernel checks for rules CUE can't express.
tags: [content, templates, cue, schemas]
status: stable
decided: "2026-11-22"
---

# Templates give starting text, and CUE schemas decide validity

## Context

The managed commands render content from templates, and the content kernel has to decide whether a body fits its type: its title, sections, word and item limits, frontmatter, labels and fields. A constraint block inside each template, or a separate JSON Schema, would put the rules in a second format beside the templates. Some rules about text shape, though, can't be seen by a schema at all, such as a section that holds only links.

## Decision

Each repository that defines content types keeps every template in a top-level `templates/` folder, with subfolders by kind, such as `templates/standard-changes/`. The managed commands render from these files, and GitHub issue forms are generated from them or checked against them. A template only creates a starting document.

A CUE schema decides validity. The kernel parses the Markdown into a normalized GitHub-flavored model of blocks and per-section metrics, adds the frontmatter and issue metadata, and validates that data against the CUE variant for the content type, unified with a policy file. Rules about text shape live in the schema. The few that CUE can't express are checked by the kernel from a short named list: links-only sections, note flooding, that a cited Request is approved, and that a retrospective's RFC exists. Each reports in the same refusal format as a schema finding.

## Options considered

- A constraint block in each template's frontmatter: rejected because templates would carry rules they can't enforce themselves.
- A JSON Schema per type: replaced by CUE, which unifies a variant with a policy file and imports the shared taxonomy.

## Consequences

A writer gets one refusal format, whether a schema or a named check found the problem. A new rule goes in the schema when CUE can express it, and the named list grows only for rules it can't. The named-list part was a reversible call under the operator's delegation, made because CUE can't see links-only content. To undo it, move those checks out of the first release. Templates and their schemas for the service-management and project-management repositories get the operator's read before they merge.
