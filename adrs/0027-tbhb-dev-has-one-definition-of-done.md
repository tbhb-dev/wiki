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

Repositories need a common standard for finished work. Each issue also needs criteria that describe what the particular change must deliver.

## Decision

The rules repository owns one definition of done and syncs it to every repository. Repositories may extend it with local requirements. Issue templates include an acceptance criteria section, and the CUE schema for each issue type enforces its structure, formatting and maximum number of criteria.

## Consequences

A contributor can start with the same completion standard in each repository. Repository-specific requirements build on the shared standard, and changes to the common requirements start in the rules repository.
