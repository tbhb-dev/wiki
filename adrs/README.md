---
type: Readme
title: Decision records
description: Which decisions this repository records, the record layout, and the index of records.
tags: [adrs, decisions, index]
status: draft
generated:
  - role: generated
    harness: codex:0.166.0
    model: openai:gpt-6.2-sol
    effort: high
    surface: exec
    session: sha-256-64:15c30f9d922d6894
    at: "2026-11-23T02:46:10Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Decision records

This public repository holds the decision records that can be stated without anything private. An ADR goes here whenever it can be sanitized without losing its meaning: host names, the tailnet, IP addresses, internal paths, account names and credentials are removed or made generic, and the rule still reads the same. Only a record that would lose its meaning goes in the private wiki repository's `adrs/` folder.

A decision record (ADR) states one decision as a rule in the present tense, with the context that led to it and what follows from it. A record makes sense to a reader of this repository alone.

## Layout

Each record is `adrs/NNNN-<slug>.md`: the next free number, zero-padded to four digits, then a kebab-case slug of the title, such as `0007-squash-merge-every-pull-request.md`. This repository numbers its own records from `0001`. [TEMPLATE.md](TEMPLATE.md) is the house template, and the format and writing rules are in the decision-records design.

## Rules

- A record never names a host, the tailnet, a vault, an internal path, an account, a private repository or a public IP address. It uses role labels such as "the Linux build host".
- A record never cites a Decisions note or links to a private repository.
- A record never quotes anyone or says who decided. Acceptance is the pull request's approval and merge, and the pull request cites the source of the decision.
- The `decided` field is the date of the decision, not of the record.
- Nobody renames a record after it merges, with one exception: the fix for a duplicate number renames the later file.

## Records

Add each new record's entry at the end of this list, as `[title](NNNN-slug.md)` and the record's description. When two pull requests add a record at once, the later one rebases and takes the next number.

- [0030. Work tracking uses GitHub Issues and Projects](0030-work-tracking-uses-github-issues-and-projects.md). GitHub Issues hold backlog items from intake onward, and organization Projects show their priority and status.
- [0031. Work issues live in the repositories responsible for them](0031-work-issues-live-in-the-repositories-responsible-for-them.md). Repository issues hold local work, and a central planning repository holds Initiatives, Epics, and Stories.
- [0032. Issues use the public repository when their text can be public](0032-issues-use-the-public-repository-when-their-text-can-be-public.md). For a public and private repository pair, issue text determines which repository receives the issue.
- [0033. Service workflows run on GitHub issues](0033-service-workflows-run-on-github-issues.md). Service workflows use issues, while reviewed pull requests preserve final reports and configuration records.
- [0034. Operator and agent requests use GitHub issues](0034-operator-and-agent-requests-use-github-issues.md). Requests make asks durable and use the Handler field, area, and suggested capabilities to route them.
- [0035. Change issues use three approval classes](0035-change-issues-use-three-approval-classes.md). Change issues come from change Requests and follow Standard, Normal, or Emergency approval rules.
- [0036. Normal Changes follow the scope of delegated authority](0036-normal-changes-follow-the-scope-of-delegated-authority.md). The responsible role, the coordinator, or the operator handles a Normal Change according to its scope and the authority it requires.
- [0037. Service issue comments are append-only work notes](0037-service-issue-comments-are-append-only-work-notes.md). Structured work notes record updates, evidence, and state transitions on service issues.
- [0038. Managed content writes use a shared command path](0038-managed-content-writes-use-a-shared-command-path.md). Service and project management commands use one content module for templates, validation, and GitHub writes.
- [0039. Content must pass its CUE schema](0039-content-must-pass-its-cue-schema.md). CUE validates normalized content and metadata, and the shared content module handles a named list of supplemental checks.
- [0040. Service and project management designs live in the public wiki](0040-service-and-project-management-designs-live-in-the-public-wiki.md). The public wiki is the home for the service-management and project-management designs.
- [0041. Project management maintains architecture decision records](0041-project-management-maintains-architecture-decision-records.md). Project management maintains the format and commands for decision records, and service management handles optional approval Requests.
- [0042. Project management defines the format for review verdicts](0042-project-management-defines-the-format-for-review-verdicts.md). Project management maintains the templates and mechanical checks for review verdicts in the shared content module.
