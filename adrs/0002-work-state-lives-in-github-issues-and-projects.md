---
type: Decision Record
title: Work state lives in GitHub Issues and Projects
description: Work items are GitHub issues gathered in org Projects, agents open them, and an issue may open whenever the work is known.
tags: [issues, projects, work-tracking]
status: stable
decided: "2026-11-22"
---

# Work state lives in GitHub Issues and Projects

## Context

Work state used to live in an unversioned notes folder: lists of open items, and tracker files with one row per step, an owner, dependencies, a state and evidence. The coordinator held much of the rest in its own context, and status moved as session messages. None of it had version history, and a lost message or a compacted context lost state. An earlier rule also kept agents from opening GitHub issues, and an issue-management design opened an issue only when someone was about to build it, so the backlog sat in documents waiting for a build slot.

## Decision

Work state lives in GitHub Issues and org Projects. Every work item is an issue, and its priority and status live in the Project's fields. Agents open, update and close issues; the rule that agents open no issues is lifted. An issue may open as soon as the work is known, not only when someone is about to build it, so the backlog is the set of open issues. The rest of the issue-management design stands: each fact has one home, verbs come before gates, and refinement happens when the work is picked up.

## Options considered

- Keep tracker files in the notes folder: rejected because they have no history and depend on every session editing rows correctly by hand.
- Open issues only at build time: rejected because the backlog then lived outside the tracker and had no priority or status.

## Consequences

Status reports read from issues and Projects rather than from messages, and a session that subscribes to the issues it cares about needs fewer cross-session messages. Until the issue templates settle and the existing trackers are imported, the tracker files in the notes folder stay the live record for the work they hold. Owning roles keep updating their own rows in place, and routine status goes into the tracker, not into messages. Security debt stays in its own tracker until it's moved into the individual private repositories rather than the central records. Opening issues freely means the Project needs triage: priority and status have to be set for each new issue.
