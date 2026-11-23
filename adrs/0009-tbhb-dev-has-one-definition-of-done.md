---
type: Decision Record
title: tbhb-dev has one definition of done, and repositories only add to it
description: One definition of done, kept in tbhb-dev/rules and synced to every repository, which may add items but never drop them; acceptance criteria follow schema-checked rules.
tags: [definition-of-done, acceptance-criteria, issues, templates]
status: stable
decided: "2026-11-22"
---

# tbhb-dev has one definition of done, and repositories only add to it

## Context

Without a shared definition, each repository and each issue states its own idea of done. Acceptance criteria then vary in structure and length, and no check can hold them to a standard.

## Decision

tbhb-dev has one definition of done, kept in tbhb-dev/rules and synced to every repository. A repository may add items to it, but never removes or weakens one.

Issue templates carry an acceptance-criteria section with rules for its structure and formatting. Each issue type has a limit on how many criteria an issue may have. The CUE schemas that validate content enforce the section, its format and the limit.

Items in the definition of done that CI enforces are marked as such and never become acceptance criteria.

## Consequences

The tbhb-dev/pm repository proposes the shape: the definition's items, the acceptance-criteria format and the per-type limits. The proposal joins the operator's read of the content templates before those templates merge.

A repository's own done criteria move into the synced definition or become additions to it. The rules repository's sync carries a change to the shared definition to every repository.
