---
type: Decision Record
title: A work item is an issue in the repository that owns the work
description: Bugs and tasks live in the repository that owns the work, Initiatives, Epics and Stories live in the central work repository, and org Projects gather them.
tags: [issues, projects, work-tracking, repositories]
status: stable
decided: "2026-11-22"
---

# A work item is an issue in the repository that owns the work

## Context

With work state moving into GitHub Issues, each item needs one home. Putting every issue in one repository would separate an item from the code, pull requests and people that act on it. Some work, though, spans repositories or doesn't belong to any single one yet. An earlier plan put cross-repository issues in a separate backlog repository only when no single repository owned the work.

## Decision

A bug or task is an issue in the repository that owns the work: tools work in the tools repository, wiki work in the wiki repository, and so on. Larger work items, meaning Initiatives, Epics and Stories, live in the central work repository, which is private. Bugs and tasks link to them as sub-issues across repositories in the same organization. Org Projects gather the issues from every repository.

The project-management repository, tbhb-dev/pm, holds the shared tooling and workflows, not other repositories' issues. Its own Issues cover only work on pm itself.

## Options considered

- One central repository for every issue: rejected because issues would sit apart from the code and pull requests they concern.
- A backlog repository only for work no single repository owns: replaced by the work hierarchy, because Initiatives and Epics span repositories by nature and need a fixed home.

## Consequences

Same-organization sub-issues link a child in one repository to a parent in another, both ways. A public child issue doesn't show its private parent to a reader who isn't signed in. When work moves to a different owning repository, its issue transfers there. The central work repository stays private, so it can describe work in terms that a public repository can't. A repository with a public and private pair picks between the two by a separate rule (record 0004).
