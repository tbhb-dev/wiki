---
type: Decision Record
title: pm owns decision records end to end
description: The pm repository owns the decision-record type, the adr command and the adrs/ rules, and service management keeps only the decision Request that can lead to a record.
tags: [adr, decisions, ownership, pm]
status: stable
decided: "2026-11-23"
---

# pm owns decision records end to end

## Context

Decision records live in `adrs/` folders in the wiki repositories, public whenever a record can be sanitized without losing its meaning. The service-management records repository had its own decisions folder and a decision template, and ownership of the command that writes records moved between the tools and service-management repositories. Decision records are architecture and planning governance, which sit between designs and Epics, not service management.

## Decision

The tbhb-dev/pm repository owns decision records end to end: the decision-record content type, the `tbhbdev adr` command (list, show, new, supersede and check, with `--private` and `--from-request N`), the `adrs/` rules and the supersede logic. Service management owns only the decision variant of a Request, which is an optional front door for a decision that needs discussion or approval first. Approving it hands off to `tbhbdev adr new --from-request N`, and the Request closes when the record merges. The tools repository supplies generic command-line and pull request check plumbing. The wiki repositories are storage.

## Options considered

- The tools repository owns the command: rejected because the command carries planning rules, not generic tooling.
- Service management owns the command beside its decision Request: rejected because records aren't service management, and the Request is only one way in.
- Separate decisions repositories: dropped, since the wiki repositories already hold designs and records.

## Consequences

The service-management decisions folder migrates into the wiki `adrs/` folders, and its decision template is reconciled with the record format. A decision can reach a record without an issue at all, through a pull request. Until `tbhbdev adr` exists, records are written by hand under the rules in the `adrs/` folder's README.
