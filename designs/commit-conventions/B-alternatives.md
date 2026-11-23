---
type: Design Appendix
title: Commit types and scopes from an allow list alternatives
description: The options the commit conventions design didn't take, and why.
tags: [commit-conventions, alternatives, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore evals -->

# Commit types and scopes from an allow list alternatives

Back to [the design](README.md).

## A single flat scope list

One list per repository, with no global or repository class, is simpler to read. It loses the ability to keep `build` and `ci` on baseline scopes, and it copies `deps`, `renovate`, and `mise` into 36, 18, and 16 repository files that then drift apart.

## `cog`'s own configuration

`cog.toml` can declare commit types, and `cog verify` already runs in every commit-msg hook. `cog` has no scope allow list and no per-type scope rule, and its file can't extend a shared baseline, so the scope check would still need its own code and configuration.

## A free scope grammar without a list

A pattern check alone, such as lowercase hyphenated words, would stop `_`, backticks, and paths. It wouldn't stop invented scopes, which is the problem the operator ruled on, so the design keeps the grammar as a schema rule on the lists and adds the list.

## A repository file that can narrow

Letting a repository remove a global scope or drop a type would let one repository diverge from the baseline unseen. The operator's ruling keeps repository files additive, and a repository that never uses a global scope loses nothing by its presence.

## A repository-level mode

A `mode` key in the repository file would let one repository enforce early or stay in `evaluate`. Letting a repository stay in `evaluate` would weaken a review control, and early enforcement buys little over a short rollout window, so `mode` is a global key only.

## Two loaders

The tools package could hold its own loader, with `check_subject` in tbhb-dev/rules holding another. Both would need the global file and the merge rule, and shared test fixtures would only slow their drift. One loader in tbhb-dev/rules, beside the files it reads, with a JSON-printing binary that tools wraps, keeps one implementation. Its cost is a subprocess call per load, which a commit or pull request command can afford.

## Requiring a scope

Requiring a scope on `feat` and `fix` would make the history easier to filter. Between 22% and 28% of those subjects carry no scope today, many of them in repositories small enough that a scope adds nothing, so the design leaves scopes optional for every type.
