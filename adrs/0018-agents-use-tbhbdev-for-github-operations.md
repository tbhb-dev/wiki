---
type: Decision Record
title: Agents use tbhbdev for GitHub operations
description: Agents use dedicated tbhbdev commands for GitHub reads and writes, and shared checks validate content before posting.
tags: [agents, github, tooling]
status: stable
decided: "2026-11-11"
---

# Agents use tbhbdev for GitHub operations

## Context

GitHub operations initiated by agents need consistent rules for identity and outgoing content. Reads also need a supported route so agents can inspect pull requests, diffs and checks without reaching for a separate client.

## Decision

Agents use a dedicated `tbhbdev` command for each supported GitHub read or write. Harness rules block direct `gh` and direct GitHub API calls. The GitHub command guard denies unmanaged writes across pull requests, issues and projects.

Commands that write GitHub content use the shared `tbhbdev.content` module. It renders text from the applicable template and checks the exact outgoing text for leaks, direct quotations of the operator, and prose violations. CUE schemas validate the rendered content and its metadata. A failed check stops the write and reports the finding so the author can fix it.

## Consequences

Each new agent operation needs a supported command. Changes to the command guard go through a reviewed pull request with tests for the affected commands. During the transition to top-level commands, existing `alpha` callers use the compatibility alias until they migrate.
