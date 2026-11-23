---
type: Decision Record
title: Content writes go only through managed commands and the content kernel
description: Every issue, comment, Project and pull request text write goes through a managed command built on the shared content kernel, and hooks block bare GitHub writes.
tags: [content, issues, hooks, kernel]
status: stable
decided: "2026-11-23"
---

# Content writes go only through managed commands and the content kernel

## Context

Agents could write issues, comments and Project items with any GitHub client, in any shape. Bare writes skip templates, checks and the audit trail, so the text on GitHub drifts from any agreed form and can leak private information. The service-management and project-management plugins both need the same write path, so building it twice would split the checks.

## Decision

Every content type has a template and a managed command, and agents write content only through those commands. Hooks in the shared command-line tool's guards block bare GitHub issue, comment and Project writes, including `gh` API mutations on them, and point agents to the managed commands. Every procedure for writing content is documented as a runbook in the wiki repositories.

Both plugins use one shared content kernel, the `tbhbdev.content` module in the public tools repository, and neither plugin calls the GitHub write APIs directly. The kernel covers issue, comment, Project item and pull request text writes. It renders from templates, sets issue types and fields, links sub-issues, and keeps the audit trail. Before any write it checks the exact rendered text with the secret scan, the quote check and the prose linter, and validates it against its content type's schema. On a finding it refuses and shows it, and it never rewords text by itself.

## Options considered

- Separate write code in each plugin: rejected because the checks would drift apart.
- Allow bare writes and check afterwards: rejected because a leak in a public repository can't be taken back.

## Consequences

Text on GitHub follows a known shape and passes the same checks everywhere. The auto-mode rule that allows Issues and Projects writes narrows to the managed commands once they exist; until then, bare writes stay allowed. The tools repository owns the kernel code, and the plugins agree on its interface. A content type without a template and a command can't be written until both exist.
