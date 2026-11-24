---
type: Design Appendix
title: Commit types and scopes evidence
description: The sources behind each claim in the commit types and scopes design, labeled proven, existing, new, or unverified.
tags: [commits, evidence, design]
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
<!-- cspell:ignore evals afenc -->

# Commit types and scopes evidence

Back to [the design](README.md).

The author took every read here on 2026-11-23. Labels work as follows:

- **Existing** claims come from the source and commit in the preceding heading.
- **Proven** claims come from a query the author ran and anyone can rerun.
- **New** claims describe this design's proposal.
- **Unverified** claims still lack a source.

This public appendix gives scope values and counts, never the repository that uses them.

## Subjects on main

### The organization count

- **Existing.** The brief that commissioned this design reported the subjects on main across 40 organization repositories over the last 60 days: 3,707 Conventional Commits subjects and 425 distinct repository-and-scope pairs. The author didn't rerun the full count.
- **Existing.** Types in that count: `chore` 1,186, `fix` 779, `feat` 727, `docs` 601, `build` 180, `test` 116, `ci` 70, `refactor` 34, `sync` 11, `perf` 2, and `revert` 1. All 11 `sync` subjects come from one repository.
- **Existing.** Subjects with no scope: `build` 146 of 180, `ci` 32 of 70, `chore` 297 of 1,186, `docs` 216 of 601, `fix` 221 of 779, `feat` 158 of 727, `test` 32 of 116, `refactor` 9 of 34, and `perf` 0 of 2.
- **Existing.** Scopes used in three or more repositories, with the number of repositories: `deps` 36 (863 subjects), `mcp` 21, `renovate` 18, `mise` 16, `tools` 10, `ci` 10, `agents` 8, `workflows` 6, `deploy` 5, then `python`, `pnpm`, `ops`, `go`, `docs`, `ansible`, and `agent` at 4, and `tokens`, `research`, `release`, `lint`, `hooks`, `evals`, `claude`, `awrel`, and `afenc` at 3.
- **Existing.** The most used single-repository scopes: `alpha` 183, `evals` 82, `designs` 77, `lint` 53, `lab` 38, `decisions` 34, `o11y` 32, `adrs` 31, `content` 30, `skills` 27, and two more at 37 and 26 that refer to one person and one network. Each stays a repository scope under the class tests.

### The author's spot check

- **Proven.** The author counted `git log main --since="60 days ago" --format=%s` over the 27 organization repositories mirrored on the build host. The count found 3,186 Conventional Commits subjects and 341 repository-and-scope pairs. The scopes in the most repositories were `deps` 23, `mcp` 18, `mise` 16, `renovate` 13, `tools` 8, `agents` 8, and `ci` 7, in the same order as the organization count.
- **Proven.** Of 2,330 scoped subjects in the spot check, 871 used one of the 16 seed scopes. By type, the seed covered `chore` 661 of 688, `fix` 108 of 433, `feat` 41 of 490, `docs` 14 of 586, `ci` 18 of 26, `build` 15 of 16, `test` 13 of 68, and `refactor` 1 of 21.
- **Proven.** Each of the 8 scoped `ci` subjects outside the seed referred to one repository's own job or tool. The one scoped `build` subject outside the seed used a backtick-wrapped `bazel`.
- **Proven.** In a sample of `tools`, `ops`, `deploy`, and `agent` subjects, each value meant different things in different repositories: another project in a design repository, and a local component elsewhere. That fails class test 2.
- **Proven.** The breaking marker appeared 6 times, all on `feat`. No subject repeated its type as its scope, such as `ci(ci)`.
- **Proven.** Of 2,746 subjects ending in a pull request number, 30 had a title part longer than 64 characters.

### Scope spellings

- **Existing.** The brief reported the scope `_` in 13 subjects, a backtick-wrapped `` `bazel` ``, and scopes that give a path or an identifier such as `e-000001`.
- **Proven.** In the spot check, 16 distinct scopes fell outside the grammar in [decision D6](decisions.md#d6-how-do-scope-spellings-outside-the-grammar-resolve): `_`, 3 backtick-wrapped values, and 12 `snake_case` values.

## Public repositories

### The rules repository at `a05c418`

- **Existing.** `lint/private/commit_messages/check_subject.py` fixes the types in code as `build|chore|ci|docs|feat|fix|perf|refactor|revert|style|test` and accepts any scope that matches `[^()\s][^()]*`. Its subject limit is 72 characters, with an exemption for the update and pin subjects that Renovate writes as `chore(deps)` and `fix(deps)`. Its code also defines the generic-summary word list.
- **Existing.** The same script passes `Revert "…"` and `Reapply "…"` unchanged, and it exits 1 on any problem.
- **Existing.** `lint/private/commit_messages.bzl` and its template define `@tbhb_rules//lint:commit_messages`, which checks each non-merge commit in `BASE..HEAD` from the workspace root. `.github/workflows/repo-checks.yml` runs it on each pull request.
- **Existing.** `.github/workflows/repo-checks.yml` scans the title and body of a pull request for secrets and doesn't check the title's form.
- **Existing.** `repo/baseline/manifest.schema.json` rejects any `source` path that starts with `/` or contains a `..` segment. Every baseline source sits under `repo/baseline/`.
- **Existing.** `repo/baseline/README.md` puts shared files under `files/` and drops the leading dot from source names. Mode `bytes` compares the whole file, and `never_write` protects paths that the repository maintains. No automation copies the baseline again after bootstrap yet.
- **Existing.** `MODULE.bazel` pins tools' source at one commit as `tbhb_rules_tbhbdev_src`, and `lint/private/review_policy.bzl` runs tools code from it.
- **Existing.** Rules' own `.pre-commit-config.yaml` runs `cog verify` and `check_subject.py` on each commit message.
- **Existing.** `repo/baseline/seed/docs/.github/workflows/ci.yml` triggers on `pull_request` with the default event types, which leave out `edited`. Bootstrap writes a seed file once, and the sync leaves it alone afterward.
- **Existing.** Rules contains no Configure Unify Execute (CUE) file and no CUE module. Its `pins/tools.json` pins the `cue` binary, and tools copies that pin.

### The tools repository at `3a026fb`

- **Existing.** `src/tbhbdev/alpha/pull_request.py` sets `MAX_TITLE = 72 - len(" (#9999)")`, which is 64, and `title_problems` checks a title's length and runs `cog verify` on it.
- **Existing.** In the same file, `read_baseline` reads shared baseline files, such as `.github/pull-request-sections.toml`, from the worktree and fails when one is missing.
- **Existing.** `src/tbhbdev/alpha/squash_merge.py`, behind `tbhbdev alpha merge`, builds the merge subject as the title plus a space and `(#N)`, and stops unless it passes `cog verify` and fits 72 characters.
- **Existing.** `src/tbhbdev/cue_runtime.py` runs only the pinned `cue` binary, with no `PATH` lookup, and a missing binary stops the check.
- **Existing.** `src/tbhbdev/schemas/` contains the CUE schemas of the content kernel, and `schemas/taxonomy.cue` lists org field values as CUE disjunctions.
- **New.** Tools' main branch lacks a function `load(section, repo_root)`. This design and the issue taxonomy design both propose it.
- **Unverified.** Tools' main branch lacks a commit command. The brief lists tools' commit command as a caller, and this design treats it as a future caller of the same target.

### The wiki at `9d9e924`

- **Existing.** [Record 0020](../../adrs/0020-pull-requests-land-through-merge-commits.md): pull requests merge through merge commits, and the first line of the merge message is the pull request title with its number.
- **Existing.** `designs/pm/B-routing-and-taxonomy.md` assigns the shared issue taxonomy design to the tools role, at `designs/issue-taxonomy/`. The brief places it at `designs/taxonomy/`.
- **Unverified.** None of the wiki mirror's branches contains the issue taxonomy design, and its loader contract is unknown. Appendix C defines the contract this design needs.
- **Existing.** The wiki's own `.pre-commit-config.yaml` checks commit messages with `cog verify` and doesn't run `check_subject`.

### GitHub Actions

- **Existing.** A title edit made with the workflow's `GITHUB_TOKEN` doesn't start an `edited` run. [Workflow triggers](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow#triggering-a-workflow-from-a-workflow)
- **Unverified.** The proposed `pr-title` queue check in option B needs a fixture to confirm that the group's first-parent range identifies every queued pull request by its second parent under the merge-commit policy. The `merge_group` event gives the base and head but carries no pull request title. [Merge group event](https://docs.github.com/en/webhooks/webhook-events-and-payloads#merge_group)
