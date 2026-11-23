---
type: Decision Record
title: 0030. Work tracking uses GitHub Issues and Projects
description: GitHub Issues hold backlog items from intake onward, and organization Projects show their priority and status.
tags: [issues, projects, tracking]
status: stable
decided: "2026-11-22"
---

# 0030. Work tracking uses GitHub Issues and Projects

## Context

People must reconcile progress when separate documents hold the backlog, the list of work to do. That setup delays recording new work until someone has time to build it.

## Decision

A GitHub issue, the record for a work item, can enter the backlog at any point, including before work can start. An organization Project, the shared GitHub board for these issues, shows their priority and status. Use the shared Priority field to show which work should come first. Record the reason for a priority change as the [routing table in the project management design](https://github.com/tbhb-dev/wiki/blob/main/designs/pm/B-routing-and-taxonomy.md#one-home-per-fact) requires. Refine the issue when someone picks it up. Keep each fact in one authoritative place.

This rule governs how to record and track work from intake onward and leaves start dates open.

## Consequences

Readers can see routine progress on the issue or Project. Keep using the current trackers until the first issue templates are ready for import. Keep using the existing list of deferred security fixes until the affected private repositories take over those entries.
