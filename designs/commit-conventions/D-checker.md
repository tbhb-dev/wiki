---
type: Design Appendix
title: Commit types and scopes checker
description: How the subject checker enforces the merged list, its arguments, messages, and exit codes, its failure modes, and its test plan.
tags: [commits, lint, testing, design]
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
<!-- cspell:ignore pnpm nope -->

# Commit types and scopes checker

Back to [the design](README.md).

Everything here is new and proposed, except where a line says the behavior exists today.

## Arguments

```text
check_subject --repo-root DIR [--title] [--evaluate | --enforce] [--list] [SUBJECT | --file PATH]
```

- `--repo-root` must be present. The checker passes it to `load("commits", DIR)` and reads nothing else from disk except `--file`.
- `--title` checks a pull request title: it applies `limits.title` instead of `limits.subject`.
- `--evaluate` and `--enforce` override the global file's `mode` for one run. The reports in plan steps 6 and 7 use `--evaluate`, and no caller in a gate passes either flag.
- `--list` prints the merged list, one line per type and per scope with its class and description, and exits 0. An agent reads it before it writes a subject.
- With `--file`, the checker reads the first line of the file, as the commit-msg stage passes it today. With neither a subject nor `--file`, it reads the first line of standard input, as the range target does today.

## The order of checks

1. A subject that starts with a `passthrough` word, a space, and a quoted subject passes.
2. Each `rewrite_prefixes` start, `unfinished` word, and over-limit length adds a problem, unless the subject starts with a `long_prefixes` entry, which lifts the length limit. These checks exist today with values in code.
3. The subject must match `type(scope)!: summary`, with the scope and `!` optional. The parser accepts any characters inside the parentheses. That lets the next step report a misspelled scope by its written form.
4. Next, the type must be a key of `types`.
5. Any scope must be on the merged list and in a class that the policy of the type accepts. Policy `none` rejects every scope.
6. Last, the summary must not be a `generic` entry after normalizing. This check exists today.

Steps 4 and 5 are the allow-list checks. In `evaluate` mode their problems print with an `evaluate:` prefix and don't change the exit code.

## Messages

Each problem is one line. The allow-list messages state the fix.

| Case | Message |
| --- | --- |
| Unknown type | `type "sync" is not allowed; use one of: build, chore, ci, docs, feat, fix, perf, refactor, revert, style, test` |
| Unlisted scope | `scope "e-000001" is not listed; drop the scope, or add it to .tbhb/commits.toml if it names a part of this repository` |
| Near miss | `` scope "`bazel`" is not listed; did you mean "bazel"? `` |
| Wrong class | `scope "designs" is a repository scope, and type "ci" takes only global scopes` |
| Scope on a `none` type | `type "style" takes no scope` |
| Title too long | `title is 65 characters; the limit is 64` |
| Configuration error | `.tbhb/global/commits.toml: missing; run the baseline sync for this repository` |

A near miss is a scope that equals a listed scope after normalizing. Normalizing strips backticks, quotes, and spaces. It then turns `_` into `-` and lowers the case. A scope that a close-match search ranks first, past a fixed cutoff, also counts as a near miss.

## Exit codes

| Exit | When |
| --- | --- |
| 0 | No problem, or only allow-list problems in `evaluate` mode, or `--list` |
| 1 | At least one problem that counts in the current mode |
| 2 | Bad arguments, or a `ConfigError` from the loader |

The range target prints its own report of each failing commit. It exits 2 when the check of any commit exits 2. A configuration error then never reads as a subject problem.

## Callers

| Caller | Invocation | Stage |
| --- | --- | --- |
| Shared CI workflow, per commit | `@tbhb_rules//lint:commit_messages -- BASE HEAD`, which passes `--repo-root "$BUILD_WORKSPACE_DIRECTORY"` | Each pull request run, as today |
| Shared CI workflow, title | `@tbhb_rules//lint:commit_subject -- --repo-root "$GITHUB_WORKSPACE" --title "$PR_TITLE"`, with the title in an environment variable | Each pull request run, new |
| `tbhbdev alpha pr check` and `pr create` | `bazelisk run @tbhb_rules//lint:commit_subject -- --repo-root <worktree> --title <title>` | Before opening, in place of `cog verify` |
| `tbhbdev alpha merge` | The same, on the title | Before merging, beside the existing 72-character test |
| A future tools commit command | The same, without `--title` | Before a commit or a push |
| The commit-msg hook | The same target, through a `bazel_env` shim, with `--file` | A later change, after enforcement |

The `commit_subject` target bundles `check_subject.py` with the loader from the pinned tools source, `tbhb_rules_tbhbdev_src`, and the pinned `cue` binary. Every caller goes through it, so a local run and CI run the same code at the repository's own rules pin.

## Failure modes

| Failure | Behavior |
| --- | --- |
| `.tbhb/global/commits.toml` is missing | Exit 2 with the sync hint. No subject passes. |
| The global file fails the schema | Exit 2 with each `cue vet` finding. |
| `.tbhb/schema/commits.cue` is missing | Exit 2. |
| The `cue` binary is missing | Exit 2. The loader never falls back to a `cue` on `PATH`. |
| `.tbhb/commits.toml` is absent | The global list alone applies. |
| `.tbhb/commits.toml` is unreadable, isn't TOML, or fails `#Repo` | Exit 2 with the file and the finding. |
| The repository file repeats a global scope | The loader keeps the global entry, and the checker prints a `notice:` line and still exits 0. |
| The checker's own code raises an unexpected error | Exit 2, never 0. The gate fails closed. |
| The subject is empty | Exit 1 with the form problem. |

## Test plan

The rules maintainer writes the checker tests under `lint/private/commit_messages/`, and the tools maintainer writes the loader tests in tools. Each fixture is a small repository root with its own `.tbhb/` files.

### Loader

- A valid global file and no repository file: the merged values equal the global values, and every scope's class is `global`.
- A repository file that adds a scope: the scope appears with class `repo`.
- A repository file that repeats a global scope: the result keeps one entry of class `global` and adds a notice.
- A missing global file, a global file with a TOML error, a global file that fails `#Global`, and a missing schema: each raises `ConfigError` naming the file.
- A repository file that's a directory, isn't readable, has a TOML error, or sets `mode`, `limits`, `types`, or `subjects`: each raises `ConfigError`.
- A scope name that breaks the grammar or runs past 24 characters, in either file: `ConfigError`.
- `schema_version = 2` in either file: `ConfigError`.
- A section name with `/` or `..`: the loader rejects it before it reads any path.
- The same fixture driven through `load("taxonomy", …)` with a schema laid out for the taxonomy, to prove the loader doesn't contain code specific to commits.

### Checker

- **Each type.** For all 11 types, a subject with no scope passes in enforce mode.
- **Each scope class.** For each type, a subject with a global scope and one with a repository scope pass or fail exactly as the type's policy says. A test fixture sets one type to `none` and one to `repo` to cover the two policies the seed doesn't use.
- **Unknown scope.** `feat(nope): add x` fails in enforce mode with exit 1, and prints an `evaluate:` line with exit 0 in `evaluate` mode.
- **Unknown type.** `sync: copy the shared files` fails with the type message.
- **Spellings.** `_`, `` `bazel` ``, `claude_code`, `src/main.py`, `a,b`, and `e-000001` each fail, and the first three get the near-miss text where a listed scope matches.
- **Breaking marker.** `feat(mcp)!: require a relay endpoint` passes, and `!` counts toward the length.
- **Lengths.** A 64-character title passes with `--title` and a 65-character one fails, while a 72-character subject passes and a 73-character one fails. A Renovate `chore(deps): update …` subject of 90 characters passes both.
- **Existing behavior.** Every case in today's `test_check_subject.py` keeps its result when the seed global file drives the checker, which proves that moving the word lists into configuration didn't change behavior.
- **Missing global file.** Exit 2 with the sync hint, in both modes.
- **Range target.** A throwaway repository with one good commit, one unlisted scope, and a missing global file exercises exits 0, 1, and 2 through `commit_messages`.
- **Mutation testing.** The checker and the loader run under the repositories' mutation test setup, and every surviving mutant gets a test or a recorded reason.
