---
type: Decision Record
title: A clone's main worktree stays clean and on main by mechanism
description: A clone's main worktree stays clean and on its default branch, enforced by the run launcher, a harness guard, a git hook and a scheduled clone audit.
tags: [git, worktrees, guards, hooks]
status: stable
decided: "2026-11-11"
---

# A clone's main worktree stays clean and on main by mechanism

## Context

Agent runs launched with a clone's main checkout as their workdir ran `prek install` there and overwrote its git hooks, the identity hooks among them. Every later run started from that clone inherits the damage. A written order to start each run in its own worktree didn't stop it, because not every launcher could meet the order.

## Decision

A clone's main worktree stays clean and on its default branch at all times, enforced in four layers:

- **Launcher.** The run launcher takes `.worktrees/<run>` as the workdir and grants write access only to that worktree and its own git directory. It refuses a workdir, workspace or write root that is a main checkout or its `.git`.
- **Harness guard.** The `tbhbdev` PreToolUse guard, for Claude and Codex, blocks file writes and git commands that move refs or change the index in a main worktree: commit, checkout or switch, reset, stash, merge, rebase, cherry-pick, `prek install` and hook installs. It allows fetch, a `--ff-only` merge or pull on the default branch, `worktree add` and `tbhbdev alpha install`.
- **Git hook.** `tbhbdev alpha install` adds a `reference-transaction` hook that aborts a HEAD move off the default branch or a new commit there. The branch may advance only to its origin counterpart or an ancestor of it.
- **Detection.** A scheduled `tbhbdev alpha clone-audit` on each host checks every clone: main worktree on the default branch, clean, at origin, and hooks matching the install. Its `--repair` option repairs hooks only.

## Consequences

For a session running as the same user, the git hook is advisory, so the harness guard is the control there. Hooks come only from `tbhbdev alpha install`, never from `prek install`.

Audit drift goes to a per-host log, which is shipped to the log store and shown on a dashboard panel. Nothing pages yet.

A legacy launcher that can't take a worktree workdir may start a run in a main checkout only under three conditions. The brief's first step creates and enters `.worktrees/<run>`. The brief forbids `prek install`, hook installs and any commit or checkout in the main checkout. The launching session checks the clone's hooks and config when the run ends.
