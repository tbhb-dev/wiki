---
type: Design Appendix
title: Commit types and scopes from an allow list test plan
description: The run-time failure modes of the commit subject check and the test cases that cover them.
tags: [commit-conventions, testing, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore evals -->

# Commit types and scopes from an allow list test plan

Back to [the design](README.md).

This plan is new, and no test exists yet. The loader cases live in tbhb-dev/rules under `config/`, the subject cases extend `lint/private/commit_messages/test_check_subject.py`, and the end-to-end cases extend the existing `commit_messages_scenario_test`. Each subject case runs once in `evaluate` and once in `enforce`, against two fixtures: a global file and a repository file.

## Failure modes

| Failure | What the user sees | Exit |
| --- | --- | --- |
| Unlisted scope, enforce | `scope "x" is not allowed for chore; allowed: …` | 1 |
| Unlisted scope, `evaluate` | The same text prefixed `evaluate:` | 0 |
| Unknown type | `type "sync" is not allowed; allowed: …` | 1 in both modes |
| Repository scope on `build` or `ci` | `scope "lab" is not allowed for build; build takes global scopes` | 1 in `enforce`, 0 in `evaluate` |
| Title over 64 characters | `title is 65 characters; the limit is 64` | 1 in both modes |
| Global file missing | `config: config/global/commits.toml is missing` on stderr | 2 in both modes |
| Schema error | The file, the field, and the schema message on stderr | 2 in both modes |
| Repository file unreadable | The file and the read error on stderr | 2 in both modes |
| `cue` missing from the `runfiles` | `config: cue is unavailable` on stderr | 2 in both modes |

A caller treats exit 2 as a stop, never as a pass. The CI step fails, the hook refuses the commit, and the tools commands stop before `gh` runs.

## Subject cases

| Case | Example | Expected in enforce |
| --- | --- | --- |
| Each of the 11 types with no scope | `build: pin cue`, through `test: cover the merge` | Pass |
| Each type with a global scope | `feat(mcp): add a server` | Pass |
| A type that takes repository scopes, with one | `fix(evals): handle an empty run` | Pass |
| `build` and `ci` with a repository scope | `ci(lab): add a job` | Fail |
| `ci` with a literal scope its repository added | `ci(evals): add a job` with the example repository file | Pass |
| An unknown scope | `chore(stuff): tidy up` | Fail, listing the allowed scopes |
| The oddities | `chore(_): x`, ``chore(`bazel`): x``, `fix(src/a.py): x`, `docs(e-000001): x` | Fail |
| A scope equal to its type | `docs(docs): fix a link` | Fail |
| An unknown type | `sync(main): pull upstream` | Fail in both modes |
| Breaking marker after the type and after the scope | `feat!: drop v1`, `feat(mcp)!: drop v1` | Pass |
| Breaking marker with an unknown scope | `feat(stuff)!: drop v1` | Fail |
| Title at 64 and 65 characters, with `--title` | Padded summaries | Pass at 64, fail at 65 |
| Subject at 72 and 73 characters | Padded summaries | Pass at 72, fail at 73 |
| A long Renovate subject and title | `chore(deps): update …` over both limits | Pass |
| Git's revert subject | `Revert "feat(stuff): x"` | Pass |
| Every check that exists today | Rewrite prefixes, `WIP`, generic summaries | Same result as today |

## Loader cases

| Case | Expected |
| --- | --- |
| Global file only | The merged section equals the global file |
| A repository file that adds scopes and a type's literal scope | Lists extend, global order first, with no duplicates |
| A repository file that repeats a global scope | The scope appears once |
| A missing global file | `ConfigError`, and exit 2 from both binaries |
| A global or repository file that fails the schema | `ConfigError` naming the file and the field |
| An unreadable or unparsable repository file | `ConfigError` |
| A repository file that sets `mode`, `max_title`, or `scopes.global` | `ConfigError` |
| A repository file that names a type the global file lacks | `ConfigError` |
| `tbhb_config commits --repo-root DIR` | The same merged section as JSON |

## End-to-end cases

- The `commit_messages` scenario test runs a throwaway repository with a fixture `.tbhb/commits.toml` and checks a range with one unlisted scope, in each mode.
- The tools tests run `pr check` and `merge --dry-run` on a draft whose title has an unlisted scope, in each mode, and on a repository with no global file.
