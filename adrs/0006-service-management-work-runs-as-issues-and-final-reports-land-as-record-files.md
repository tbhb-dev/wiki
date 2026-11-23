---
type: Decision Record
title: Service-management work runs as issues, and final reports land as record files
description: Incidents, Problems, Requests and Changes run their workflows as issues, while final reports and configuration items land as record files by pull request with no shared index in the diff.
tags: [service-management, issues, records, workflow]
status: stable
decided: "2026-11-22"
---

# Service-management work runs as issues, and final reports land as record files

## Context

The private service-management records repository holds incidents, problems, requests, changes and configuration items. Some of these have a lifecycle with states, and some are finished documents that should be versioned and reviewed. An earlier design committed records straight to the main branch through a bypass, which skipped review. A shared index file edited in every pull request would make concurrent pull requests conflict and force rebases.

## Decision

Anything with a workflow is an issue, and GitHub Actions apply labels and enforce its state machine.

- An incident runs as an issue until it closes. Its final incident report then lands by pull request in `incidents/`, with CI and agent review.
- A problem runs as an issue until it's resolved. Its final problem report then lands by pull request in `problems/`, with CI and agent review.
- Requests and Changes are issues only. A Change's templates are files in the repository, but the Change itself has no record file.
- Configuration items are files in `cmdb/`, added and changed by pull request with CI and agent review.

No record folder requires a pull request to be up to date with the main branch, and no pull request edits a shared index. An Actions workflow rebuilds each folder's `index.json` on push to the main branch, using a dedicated GitHub App whose credential is a repository Actions secret.

## Options considered

- Direct commits of records to the main branch: rejected because records then skip review.
- An index file edited in each pull request: rejected because concurrent pull requests would conflict over it.

## Consequences

Record pull requests don't conflict with each other and don't need rebases. The index lags the main branch by one workflow run. The App holds write access only for the index workflow, which runs in Actions with no webhook receiver. Decision records aren't service-management records: they live in the wiki repositories (record 0012).
