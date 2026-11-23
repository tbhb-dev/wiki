---
type: Design Appendix
title: Commit types and scopes from an allow list evidence
description: The subject survey and the code read for the commit conventions design, each claim labeled.
tags: [commit-conventions, evidence, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore afenc awrel tailnet evals -->

# Commit types and scopes from an allow list evidence

Back to [the design](README.md).

Each claim carries a label: **existing** for merged code or data read on 2026-11-23, **new** for what this design proposes, and **unverified** for what no one has checked yet. This appendix cites tbhb-dev/rules at `22cd74b` and tbhb-dev/tools at `001505b`.

## The subject survey

**Existing.** A survey of the subjects on `main` across 40 organization repositories, over the 60 days before 2026-11-23, found 3,707 Conventional Commits subjects and 425 distinct pairs of repository and scope. This public appendix names scope values and counts only, never the repository that uses them.

| Type | Subjects | With no scope | Share with no scope |
| --- | --- | --- | --- |
| `chore` | 1,186 | 297 | 25% |
| `fix` | 779 | 221 | 28% |
| `feat` | 727 | 158 | 22% |
| `docs` | 601 | 216 | 36% |
| `build` | 180 | 146 | 81% |
| `test` | 116 | 32 | 28% |
| `ci` | 70 | 32 | 46% |
| `refactor` | 34 | 9 | 26% |
| `sync` | 11 | not counted | not counted |
| `perf` | 2 | 0 | 0% |
| `revert` | 1 | not counted | not counted |

`sync` isn't an allowed type, and all 11 of its subjects come from one repository. `style` has no subjects in the window.

Scopes used in three or more repositories, with the number of repositories:

- `deps` 36, with 863 subjects
- `mcp` 21, `renovate` 18, `mise` 16, `tools` 10, `ci` 10
- `agents` 8, `workflows` 6, `deploy` 5
- `python`, `pnpm`, `ops`, `go`, `docs`, `ansible`, and `agent`, 4 each
- `tokens`, `research`, `release`, `lint`, `hooks`, `evals`, `claude`, `awrel`, and `afenc`, 3 each

The most used scopes that appear in one repository only, by subject count: `alpha` 183, `evals` 82, `designs` 77, `lint` 53, `lab` 38, `tony` 37, `decisions` 34, `o11y` 32, `adrs` 31, `content` 30, `skills` 27, and `tailnet` 26. `evals` and `lint` appear in both lists because one repository uses each far more than the others.

The oddities: the scope `_` in 13 subjects, a backtick-wrapped `` `bazel` ``, and scopes that name a path or an identifier such as `e-000001`.

## The checks today

- **Existing.** `lint/private/commit_messages/check_subject.py` in tbhb-dev/rules fixes the types `build`, `chore`, `ci`, `docs`, `feat`, `fix`, `perf`, `refactor`, `revert`, `style`, and `test` in a regular expression, and accepts any parenthesized scope without spaces or nested parentheses. It allows `!` before the colon.
- **Existing.** The same file sets `MAX_LENGTH = 72`, exempts `chore` and `fix` subjects scoped `deps` or `deps-dev` whose summary starts with `update` or `pin` from the length limit, passes the git `Revert "…"` and `Reapply "…"` subjects, and refuses rewrite prefixes, `WIP`, and a fixed set of generic summaries. Every one of those value lists sits in the code.
- **Existing.** It reads a message from stdin or from a file named by its first argument, prints one problem per line, and exits 1 on any problem and 0 otherwise. It has no exit code for a configuration error, because it reads no configuration.
- **Existing.** `@tbhb_rules//lint:commit_messages` runs it on each non-merge commit between a base and a head, and the rules `repo-checks.yml` workflow runs that target on every pull request.
- **Existing.** The same workflow scans the pull request title for secrets, but no CI step checks the title's form.
- **Existing.** The rules repository's own commit-msg hook runs `cog verify` and then `check_subject.py`. The wiki's and tools' commit-msg hooks run `cog verify` only.
- **Existing.** `tbhbdev alpha pr check` in tbhb-dev/tools sets `MAX_TITLE = 72 - len(" (#9999)")`, which is 64, and runs `cog verify` on the title. `tbhbdev alpha merge` checks the final subject, title plus a space and `(#N)`, against `cog verify` and 72 characters.
- **Existing.** tools has no commit command today. Commits reach the check through the commit-msg hook.
- **Existing.** tbhb-dev/rules pins the `cue` binary as `tbhb_rules_cue` but has no `.cue` file and no `config/` directory yet.

## The loader and the taxonomy design

- **Unverified.** No `load(section, repo_root)` function exists in tools at `001505b`, and no `designs/taxonomy/` design exists on this repository's `main` or its open branches on 2026-11-23. This design fixes the loader shape in [D. Configuration](D-configuration.md) so the taxonomy design can adopt it.
- **Unverified.** Whether `bazel_env` can expose a Python binary from tbhb-dev/rules with its `runfiles` intact. Each shim it exposes today is a prebuilt binary, such as `cog`, `gitleaks`, or `vale`. Plan step 1 answers it.

## What this design adds

- **New.** The global and repository files, the schema, the single loader, the scope grammar, the per-type classes, `--title`, `--subject`, and `--repo-root`, exit code 2, `evaluate` mode, and the CI title step.
