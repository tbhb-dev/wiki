---
type: Decision Record
title: tbhb-dev has one definition of done
description: The rules repository owns a shared definition of done and issue types define the shape of their acceptance criteria.
tags: [acceptance-criteria, issues, rules]
status: stable
decided: "2026-11-22"
---

# tbhb-dev has one definition of done

## Context

tbhb-dev repositories need a common standard for completed work, called the definition of done. Each issue needs acceptance criteria that describe what its change must deliver. The rule covers the shared completion standard and the form of acceptance criteria, leaving the specific criteria to each issue.

## Decision

The [tbhb-dev/rules repository](https://github.com/tbhb-dev/rules/blob/main/README.md) maintains one definition of done. It syncs the standard to every tbhb-dev repository. Repositories may extend it with local requirements. Issue templates provide a section for acceptance criteria. CUE is a language for defining data constraints. A CUE schema for each issue type enforces the section's structure, formatting and maximum number of criteria.

## Consequences

A contributor can start with the same completion standard in each repository. Repository-specific requirements build on the shared standard, and changes to the common requirements start in the rules repository.
