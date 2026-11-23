---
type: Design Appendix
title: Commit types and scopes alternatives
description: The policies, file homes, and check mechanisms the commit types and scopes design didn't take, and why.
tags: [commits, alternatives, design]
status: draft
parent: README.md
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-opus-5-5
    surface: cli
    at: "2026-11-23T15:44:11Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore Cocogitto -->

# Commit types and scopes alternatives

Back to [the design](README.md).

## Policies

- **`feat` and `fix` take only repository scopes.** Rejected. Renovate writes `fix(deps)`, and repositories add features to their Model Context Protocol (MCP) servers and agent files. In the spot check, 149 scoped `feat` and `fix` subjects used a seed scope.
- **`build`, `ci`, `docs`, and `chore` are scope-free.** Rejected. Writers already scope 34 of 180 `build` and 38 of 70 `ci` subjects, and most Renovate subjects use `chore(deps)`. A scope-free policy throws that signal away.
- **`docs` takes only global scopes.** Rejected. In the spot check, 572 of 586 scoped `docs` subjects referred to a repository's own part, such as `designs` or `adrs`.
- **A required scope for `feat` and `fix`.** Rejected. A writer without a fitting scope invents one, and invention is the problem the list solves. 158 `feat` and 221 `fix` subjects read well with no scope.
- **Only a global list.** Rejected. A global list large enough for each repository's parts would expose private repository names in a public file, and each repository's scopes would wait on a rules release.

## Homes

- **The brief's `config/global/` and `config/schema/` at the rules root.** Changed. The baseline manifest rejects a source outside `repo/baseline/`. Files there couldn't reach other repositories through the sync.
- **A `[commits]` table in `.tbhb-repo.toml`.** Rejected. That file declares a repository's kind and visibility under its own JSON Schema in rules. Adding a section checked by Configure Unify Execute (CUE) gives one file two schema languages and two maintainers.
- **The global file loaded from the rules module through Bazel, with no copy in the worktree.** Rejected. The loader for the issue taxonomy runs outside Bazel, and one loader needs one place to look. A worktree copy also matches how tools reads the file of pull request sections today.
- **The schema inside the tools package, beside the content schemas.** Rejected. The schema and the global file change together, and they share one source and one sync. With both in the worktree, the loader doesn't depend on anything else.

## Mechanisms

- **`cog.toml` scopes and types in each repository.** Rejected. Pinned Cocogitto 7.0.0 supports a top-level `scopes` list. Its tagged tests show that `cog verify` accepts a listed scope and rejects an unlisted scope. The checker enforces the merged list and each type's scope policy. Existing `cog verify` callers keep their syntax checks without adding a second scope list. [Cocogitto 7.0.0 verification tests](https://github.com/cocogitto/cocogitto/blob/7.0.0/crates/cocogitto/tests/cog_tests/verify.rs#L63-L111)
- **A second checker in tools for pull request titles.** Rejected. Separate checkers drift, and today the 64-character title rule already sits apart from the 72-character subject rule. One target serves both.
- **Taking scope lists from the base branch.** Under [decision D7](decisions.md#d7-which-commits-configuration-does-a-pull-request-check-read), the head supplies the scope lists so a pull request can add and use a scope in the same change. The base supplies only its mode.
- **Enforcing at once with no `evaluate` period.** Rejected. About 1,500 scoped subjects in the spot check used a scope outside the seed. Day one would fail most pull requests until each repository wrote its file.
- **Rewriting old subjects to the new list.** Rejected. Rewriting main breaks signatures, merges, and existing clones, and readers can still follow the old subjects.
