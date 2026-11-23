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

A Git clone is a local copy of a remote repository. Its main worktree, the checkout created with the clone, gives contributors an unmodified copy of the default branch, `main`. An agent run there can leave local changes or replace the clone's shared hooks, which run on Git events. The rule protects the main checkout and leaves work on other branches to separate checkouts.

The launcher starts agent runs. The harness guard enforces what the software running the agent may write. A Git directory stores Git metadata. References, or refs, record branch positions. The index is Git's staging area for the next commit. A fast-forward advances a branch without rewriting its history. The managed hook installer sets up the approved Git hooks.

## Decision

The launcher starts agents in dedicated worktrees. Each run can write to its worktree and Git directory. The launcher refuses the main worktree and its Git directory as workspace or writable roots. The guard blocks file writes, ref moves and index changes in the main worktree. It permits fetches, fast-forward-only default-branch updates, worktree creation and the managed hook installer.

The reference-transaction hook runs when Git updates refs. It keeps the checkout on the default branch. That branch can advance only to the remote branch's latest fetched commit or one of its ancestors. The harness guard enforces the rule for runs sharing the checkout's user account.

## Consequences

A scheduled clone audit checks the checked-out branch, local file changes, agreement with the remote branch and installed hooks. It reports departures from the required state. It can restore the managed hooks. Contributors create and switch branches in separate worktrees.
