---
type: Decision Record
title: 0032. Issues use the public repository when their text can be public
description: For a public and private repository pair, issue text determines which repository receives the issue.
tags: [issues, privacy, routing]
status: stable
decided: "2026-11-22"
---

# 0032. Issues use the public repository when their text can be public

## Context

A project with public and private repositories needs a default issue destination and a way to keep sensitive operational details private.

This rule covers issue creation and transfer within a repository pair and leaves the choice of [which repository is responsible for the work](0031-work-issues-live-in-the-repositories-responsible-for-them.md) outside its scope.

## Decision

Open the issue in the public repository when its text can be public. Use the paired private repository when the text needs details barred from publication. At runtime, the [project management commands](0038-managed-content-writes-use-a-shared-command-path.md) find the counterpart from the organization's rules for naming repository pairs. They check the exact text GitHub would receive before a public write. If the check finds a private detail, the commands refuse the public destination. They offer the private destination for that finding. If the check finds a secret such as a credential, the commands refuse the write.

Use GitHub's built-in issue transfer to move an existing issue between the repositories. Run the same scan first. Keep the author's text intact.

## Consequences

Public work remains visible by default. A scan result explains why the public destination is unavailable. Each item has one issue across the pair.
