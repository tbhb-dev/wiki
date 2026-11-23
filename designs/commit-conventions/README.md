---
type: Design Draft
title: Commit types and scopes from an allow list
description: How tbhb-dev repositories check commit subjects and pull request titles against one merged allow list of types and scopes.
tags: [commit-conventions, commits, config, lint, design]
status: draft
owner: Rules maintainer
updated: "2026-11-23"
decisions_open: 3
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore afenc awrel tailnet evals -->

# Commit types and scopes from an allow list

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Rules maintainer | 2026-11-23 | 3 open decisions in [decisions.md](decisions.md), about 10 minutes |

## Summary

Commit subjects fix their type but accept any scope, so agents invent scopes and the history stops sorting cleanly. The operator ruled on 2026-11-23 that types and scopes come from an allow list.

This design puts every type, scope, and per-type rule in configuration: a global file in the rules baseline, which each repository extends with its own file and never narrows. The existing subject check reads the merged list, and the same check runs on commit messages, pull request titles, and merge subjects.

For the operator, an unlisted scope stops a commit or a pull request once the check leaves its `evaluate` mode. Until then it only reports.

## Proposal

Once built, the design adds these pieces:

- **The global file**, `config/global/commits.toml` in tbhb-dev/rules. It lists the types, the global scopes, the scope classes each type accepts, and the length limits. The rules maintainer owns it, and a rules release carries it to each repository at that repository's pinned rules version.
- **A repository file**, `.tbhb/commits.toml`, in any repository that has scopes of its own. It adds that repository's scopes and may widen what a type accepts. The repository's own maintainer owns it.
- **The schema**, `config/schema/commits.cue` in tbhb-dev/rules, which validates both files and the merged result.
- **The loader**, one implementation in tbhb-dev/rules that reads the global file, then the repository file, validates both, and merges them. The tools package's `load(section, repo_root)` calls it, and so does the issue taxonomy design for its own section.
- **The subject check**, `check_subject` in tbhb-dev/rules, which enforces the merged list. The commit-msg hook, the continuous integration (CI) commit check, a new CI title check, and the tools pull request and merge commands all call it.

Nothing new runs as a service. Each check runs in the process that calls it, reads files at the repository's checkout, and keeps no state.

## How it works

### Scope classes

A scope belongs to one of two classes, and the class decides which file lists it.

| Class | Listed in | What makes a scope this class | Seed examples |
| --- | --- | --- | --- |
| Global | `config/global/commits.toml` | It names a tool, a dependency channel, or a baseline concern whose meaning doesn't change between repositories. | `deps`, `renovate`, `mise`, `mcp`, `ci`, `agents` |
| Repository | `.tbhb/commits.toml` | It names a component, directory, or product area of one repository. | One repository's `evals`, `designs`, or `lab` |

Usage in three or more repositories is evidence for the global class, not the test. A scope three repositories use for three different things stays in each repository's file. [C. Seed lists](C-seed-lists.md) gives the proposed global list, the scopes left to repository files, and the oddities.

Every scope matches the grammar `^[a-z][a-z0-9]*(-[a-z0-9]+)*$`, lowercase words joined by hyphens. The schema enforces it on the lists, so no list can hold a scope such as `_`, a backtick-wrapped name, or a path.

### Per-type allow lists

Each type in the global file names the scope classes it accepts, and may list extra literal scopes. A subject with no scope passes for every type, and no type requires one. The recommendation, open as D3 in [decisions.md](decisions.md):

| Type | Accepts | Subjects with no scope today |
| --- | --- | --- |
| `feat`, `fix`, `docs`, `chore`, `test`, `refactor`, `perf`, `style`, `revert` | Global and repository scopes | 22% to 36% for the types with more than 100 subjects |
| `build`, `ci` | Global scopes only | 81% for `build`, 46% for `ci` |

No type is scope-free, and none is open to unlisted scopes. "Open" in this design always means any scope on the merged list. A repository widens one type for itself by adding a class or a literal scope to that type in its own file.

### The merged configuration

The loader follows the same steps for every section, so `commits` and the issue taxonomy's section share one shape:

1. Read `config/global/<section>.toml` from the rules `runfiles`. A missing or unreadable file fails closed.
1. Read `<repo_root>/.tbhb/<section>.toml`. A missing file means the repository adds nothing, and an unreadable or unparsable one fails closed.
1. Check the global file against `#Global`, the repository file against `#Repo`, and the merge against `#Merged`, all from `config/schema/<section>.cue`. Any schema error fails closed.
1. Merge: each list becomes the union of the global and repository values, global order first, without duplicates. Scalars and exemption lists come from the global file only, because `#Repo` rejects them.

A repository can only add scopes, add classes or literal scopes to a type, and add words to the generic-summary list. It can't add a type, remove anything, raise a length limit, or change the mode. [D. Configuration](D-configuration.md) gives both files, the schema, and the merge rule in full.

### The subject check

`check_subject` reads a subject and a repository root, loads the `commits` section, and prints one problem per line.

```text
check_subject [--title] [--repo-root DIR] [--subject TEXT | FILE]
```

| Exit | Meaning |
| --- | --- |
| 0 | The subject passes, or it fails only the scope allow list while the mode is `evaluate`. |
| 1 | The subject has at least one problem. |
| 2 | Usage error, or the configuration failed closed. |

The checks run in order: the git `Revert "…"` and `Reapply "…"` subjects pass as today, then rewrite prefixes, `WIP`, length, the type, the scope against the type's merged allow list, a scope that repeats its type such as `docs(docs)`, and generic summaries. Every type accepts the `!` breaking marker after the type or the scope, and the marker counts toward the length. An unknown scope prints the scopes the type accepts, nearest first.

### Pull request titles

A squash merge turns the title into the subject, so a title gets the same check with `--title`. The limit is `max_title = 64`, the 72-character subject limit less the space and `(#N)` suffix the merge appends, which `tbhbdev alpha pr check` already enforces. The tools merge command checks the final subject, suffix included, against 72.

These callers run the check, the local ones before a push and the CI ones on every pull request:

| Caller | What it checks | Today |
| --- | --- | --- |
| The commit-msg hook in each repository's hook configuration | The commit subject | `cog verify`, plus `check_subject` in tbhb-dev/rules only |
| The CI commit check, `@tbhb_rules//lint:commit_messages` | Each non-merge commit on the pull request | `check_subject` without scopes |
| A new CI title step | The pull request title, with `--title` | A secret scan only |
| `tbhbdev alpha pr check`, `create`, and `edit` | The draft title, with `--title` | `cog verify` and 64 characters |
| `tbhbdev alpha merge` | The merge subject | `cog verify` and 72 characters |

Tools reaches the check through the `bazel_env` shim directory, `.bazel_env/bin/check_subject`, the same path its hooks use for `cog`. The `cog verify` calls stay for the full message grammar.

### Rollout and history

The global file's `mode` starts at `evaluate`. In that mode a scope that fails the allow list prints as `evaluate: …` and exits 0, while every check that exists today keeps failing as it does now. When 14 days pass with no `evaluate` report that a repository file didn't then fix, the rules maintainer sets `mode = "enforce"` in a rules release.

The check runs only on new commits and titles. Nothing rewrites or rechecks the history on `main`, and the 13 subjects with the scope `_` and the 11 with the type `sync` stay as written.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| The seed global list misses a scope that many repositories need | Commits fail across repositories once enforced | The `evaluate` mode reports every miss first, and a rules release adds the scope | new |
| An open pull request spans the switch to enforce | Its existing commits fail the CI commit check | The rules release notes the switch, and the author rewords before merging, as for any subject failure today | existing |
| `bazel_env` can't expose a Python binary with its `runfiles` | Tools can't call the check or the loader | Plan step 1 proves it, and the fallback is `bazelisk run @tbhb_rules//lint:check_subject` | unverified |
| Renovate titles exceed 64 characters | The new CI title step fails dependency pull requests | The global `length_exempt` prefixes cover titles as they cover subjects today | existing |
| The issue taxonomy design defines a different loader shape | Two loaders drift | This design fixes the shape in D, and the taxonomy design adopts it or says why not | unverified |
| A repository lists a scope that belongs in the global list | Duplicate meanings across repositories | The rules maintainer reviews repository files during the `evaluate` window and promotes shared scopes | new |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | Schema, global file in `evaluate`, and the loader in tbhb-dev/rules, plus a `bazel_env` spike for a Python binary | `bazelisk test //config/...` passes the loader cases in [E](E-test-plan.md), and `.bazel_env/bin/check_subject --help` runs in a consumer repository | Rules maintainer |
| 2 | `check_subject` reads the merged list, with `--title`, `--subject`, and `--repo-root` | `bazelisk test //lint/private/commit_messages/...` passes every case in [E](E-test-plan.md) | Rules maintainer |
| 3 | The CI title step and the baseline commit-msg hook call `check_subject` | A pull request with an unlisted scope shows an `evaluate:` line in the CI log and passes | Rules maintainer |
| 4 | `tbhbdev alpha pr` and `merge` call the check, and `load(section, repo_root)` wraps the rules loader | The tools tests show an unknown scope reported in `evaluate` and refused in `enforce` | Tools maintainer |
| 5 | Each repository with its own scopes adds `.tbhb/commits.toml` | That repository's `evaluate` reports are empty for new pull requests | Each repository's maintainer |
| 6 | `mode = "enforce"` in a rules release | A test commit with an unlisted scope exits 1 in a repository at that rules version | Rules maintainer |

## Appendices

- [Decisions](decisions.md): 3 open, then the decided record.
- [A. Evidence](A-evidence.md): the subject survey and the code read for this design, each labeled.
- [B. Alternatives considered](B-alternatives.md): the options not taken, and why.
- [C. Seed lists](C-seed-lists.md): the proposed global scopes, the scopes left to repository files, and the oddities.
- [D. Configuration](D-configuration.md): both files, the schema, the merge rule, and the loader's failure cases.
- [E. Failure modes and test plan](E-test-plan.md): what can fail at run time and the cases that cover it.
