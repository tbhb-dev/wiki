---
type: Decision Record
title: Clone main worktrees stay clean and on main
description: The launcher, command guard, Git hook and scheduled audit keep each clone's main worktree clean and on its default branch.
tags: [agents, git, worktrees]
status: stable
decided: "2026-11-11"
---

# Clone main worktrees stay clean and on main

## Context

The main worktree gives contributors a clean copy of the remote default branch. An agent run launched there can dirty that checkout or replace the clone's shared hooks.

## Decision

The launcher runs agents in dedicated worktrees and uses the run worktree and its own Git directory as the writable workspace. It refuses the main worktree or its Git directory as a workspace or writable root. The harness guard blocks file writes and Git commands that move refs or change the index in the main worktree. It permits fetches, fast-forward-only updates on the default branch, worktree creation and the managed hook installer.

The installed Git reference-transaction hook keeps the checkout on the default branch. It allows that branch to advance only to the fetched remote default tip or one of its ancestors. The harness guard provides enforcement for runs that share a user account with the checkout.

## Consequences

A scheduled clone audit checks the branch, working tree, remote alignment and installed hooks. It reports drift and can restore the managed hooks. Contributors create and switch branches in separate worktrees.
