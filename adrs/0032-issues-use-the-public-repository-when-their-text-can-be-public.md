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

Open the issue in the public repository when its text can be public. Use the paired private repository when the text needs details barred from publication. The [project management commands](0038-managed-content-writes-use-a-shared-command-path.md) find counterparts at runtime using the organization's rules for naming repository pairs and scan the exact text GitHub would receive before a public write. If they find a private detail, they offer only the private destination. They refuse writes containing secrets such as credentials.

Run the same scan first. Keep the author's text intact. Use GitHub's built-in transfer from public to private. For private to public, open a new public issue with the scanned text, link the old issue to it, and close the old issue.

## Consequences

Public work remains visible by default. A scan result explains why the public destination is unavailable. Each item has one open issue across the pair.
