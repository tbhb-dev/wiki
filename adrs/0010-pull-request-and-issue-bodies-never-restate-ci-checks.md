---
type: Decision Record
title: Pull request and issue bodies never restate CI checks
description: Pull request bodies, issue bodies and acceptance criteria carry no claims about tests, gates, lint or coverage; CI status on the head is the record.
tags: [pull-requests, issues, ci, templates]
status: stable
decided: "2026-11-22"
---

# Pull request and issue bodies never restate CI checks

## Context

A body that says the tests pass goes stale as soon as the head changes, and it repeats a status GitHub already shows on the head. When the claim and the check disagree, a reader can't tell which to trust.

## Decision

Pull request (PR) bodies, issue bodies and acceptance criteria never claim that tests pass, report gate results, claim lint or coverage, or list CI-checked items as a checklist. CI status on the exact head is the record.

Items in the definition of done that CI enforces are marked as enforced by CI and never become acceptance criteria.

## Consequences

The PR draft template and schema in tbhb-dev/rules, and the content templates and checks in `tbhbdev.content`, enforce the rule. A body's verification section gives the commands run and what they showed, minus what CI already reports.

The checks that catch restatements are best-effort text filters. A filter that refuses honest text breaks basic use, so it blocks the filter's merge. A restatement the patterns miss is tech debt, logged with the missed text as a test fixture for a follow-up.
