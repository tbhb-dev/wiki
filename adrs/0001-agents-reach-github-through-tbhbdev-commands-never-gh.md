---
type: Decision Record
title: Agents reach GitHub through tbhbdev commands, never gh
description: Agents never run gh or call the GitHub API directly; each GitHub operation they need is a tbhbdev command, and harness guards deny the rest.
tags: [github, agents, guards, tbhbdev]
status: stable
decided: "2026-11-11"
---

# Agents reach GitHub through tbhbdev commands, never gh

## Context

Agents act on GitHub all day: they open, edit, close and merge pull requests (PRs), read checks and diffs, and post reviews. Through `gh` or raw API calls, each of those is a write nobody scanned or a read nobody budgeted. An earlier wrapper that scanned text before a GitHub write and refused identities outside an allowlist showed that a command can enforce what a bare client can't.

A later read of the guard hook found that it denied only `gh pr create`, `edit` and `merge`. Close, reopen, comment, ready, review, every issue write and every `gh api` write still went through the hook and every deny list.

## Decision

Agents never run `gh` directly and never send raw HTTP requests to the GitHub API. Every GitHub operation an agent needs, reads included (PR view, checks, diffs), is its own `tbhbdev` command, which can scan text, check identity and budget calls before it reaches GitHub.

Enforcement is part of the rule. The agent harness settings deny `gh` and raw access to the GitHub API, and the `tbhbdev` guard hook denies every `gh` write, not a chosen few. A change to the guard goes by PR with a corpus test of command forms and the heavier review, never as a direct install.

## Options considered

- Deny only the writes judged risky (create, edit, merge): rejected because every other write slipped past.
- Let agents run `gh` behind a text scanner: rejected because reads need the same identity checks and call budget as writes.

## Consequences

The tbhb-dev/tools repository inventories the `gh` operations agents use and maps each to a `tbhbdev` command. The commands sit under `tbhbdev alpha` until that namespace is removed, and every caller moves in the same rollout.

Content writes such as issue text, comments and PR text go through the shared `tbhbdev.content` module, which checks the exact text before it's written. The operator's own user settings carry the same denies, and the operator changes them.

Wrapper forms that a hook can't see into stay documented as known gaps, and server-side rulesets remain the boundary for protected branches. A command a guard blocks is never handed to another agent to run instead. Undo: revert the guard and settings PRs.
