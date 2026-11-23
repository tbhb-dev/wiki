---
type: Decision Record
title: The org issue taxonomy adds Problem, Request and Change
description: The org has ten issue types, the original seven plus Problem, Request and Change, with one Severity definition and the shared fields and label families.
tags: [issues, taxonomy, service-management]
status: stable
decided: "2026-11-22"
---

# The org issue taxonomy adds Problem, Request and Change

## Context

The organization already had seven issue types, three org fields and shared label families, carried over from an earlier taxonomy file. Service management needs kinds of record the seven don't cover: an underlying cause behind failures, an ask for a decision or an action, and a change to a running system.

## Decision

The org adds three issue types to the existing seven. A Problem records the underlying cause of incidents or of a recurring failure. A Request asks the operator, or another role, for a decision, an action, access or information (record 0005). A Change records a change to a running system (record 0008). The administrator role creates the types, with descriptions written to match the existing seven.

The earlier taxonomy file is the base: the ten types, the three org fields, the shared label families, and its placement rule that nothing is recorded twice. Severity has one definition, the org Severity field's, with values SEV1 to SEV3. A Problem's priority uses the org Priority field (record 0001). The taxonomy is written as CUE definitions that every content type's schema imports.

## Consequences

Every repository uses the same types and fields, so one org Project can show all of them. Adding a type or field is an org setting change for the administrator role. Because severity has one definition, an incident's severity and a Problem's impact read the same everywhere.
