---
type: Decision Record
title: 0038. Managed content writes use a shared command path
description: Service and project management commands use one content module for templates, validation, and GitHub writes.
tags: [content, service-management, projects]
status: stable
decided: "2026-11-22"
---

# 0038. Managed content writes use a shared command path

## Context

Issues, comments, [Project](0030-work-tracking-uses-github-issues-and-projects.md) entries, and pull requests need a common way to check proposed writes and record what changed.

The shared path governs the content that management commands write to GitHub and leaves workflow approvals to the rules for each kind of work.

## Decision

Each kind of [service record](0033-service-workflows-run-on-github-issues.md) has a template and a managed command that enforces its content rules. Group templates by kind in a directory at the repository root. Commands render text from those files. Use the files to generate or check GitHub issue forms. Document the procedures in runbooks, the guides for carrying out routine work.

The service and project management command extensions share the `tbhbdev.content` module for GitHub writes. It checks the exact rendered text. Gitleaks scans for secrets. Vale checks prose rules. The module detects quotations. It validates the record against its [schema](0039-content-must-pass-its-cue-schema.md). Any finding stops the write and tells the author why. On success, the module writes issue fields and links. It logs the operation.

## Consequences

As managed commands become available, agent hooks, the checks that run before actions, and write permissions route issue, comment, and Project changes through them. Both extensions use the same checks. Authors revise refused content before trying again.
