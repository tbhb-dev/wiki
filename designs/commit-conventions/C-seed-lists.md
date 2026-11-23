---
type: Design Appendix
title: Commit types and scopes from an allow list seed lists
description: The proposed global scopes, the scopes left to repository files, and the oddities, seeded from the subjects in use.
tags: [commit-conventions, scopes, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore afenc awrel tailnet evals -->

# Commit types and scopes from an allow list seed lists

Back to [the design](README.md).

The counts come from the survey in [A. Evidence](A-evidence.md#the-subject-survey). A scope is global when it names a tool, a dependency channel, or a baseline concern whose meaning doesn't change between repositories. Usage across repositories is supporting evidence only.

## Proposed global scopes

The operator reviews this list as D2 in [decisions.md](decisions.md).

| Scope | Repositories using it | What it names everywhere |
| --- | --- | --- |
| `deps` | 36 | Runtime dependency updates by Renovate or by hand |
| `deps-dev` | not counted | Development dependency updates, already exempt from the length limit today |
| `mcp` | 21 | Model Context Protocol server configuration |
| `renovate` | 18 | The Renovate configuration |
| `mise` | 16 | The mise tool pins |
| `tools` | 10 | Developer tool pins and their shims |
| `ci` | 10 | CI workflows and their configuration |
| `agents` | 8 | Agent instructions and harness configuration |
| `docs` | 4 | Documentation tooling and site configuration |
| `python` | 4 | The Python toolchain and its project files |
| `go` | 4 | The Go toolchain and its module files |
| `pnpm` | 4 | The pnpm toolchain and its workspace files |
| `ansible` | 4 | The Ansible toolchain and its collections |
| `release` | 3 | The release process and its configuration |
| `lint` | 3 | Lint configuration and its gates |
| `hooks` | 3 | Git hook configuration |
| `bazel` | not counted | The Bazel module, version, and settings files, seen only in the backtick form |

`bazel` enters the list because every repository builds with Bazel, though the survey saw it only as an oddity. The operator can strike it in answering D2.

## Scopes left to repository files

| Scope | Repositories using it | Why not global | Suggested global scope instead |
| --- | --- | --- | --- |
| `workflows` | 6 | It overlaps `ci`. | `ci` |
| `deploy` | 5 | Its target differs by repository. | none |
| `ops` | 4 | Its meaning differs by repository. | none |
| `agent` | 4 | It overlaps `agents`. | `agents` |
| `claude` | 3 | It names one harness, which `agents` covers. | `agents` |
| `tokens`, `research`, `evals`, `awrel`, and `afenc` | 3 each | Each names a product area. | none |

A repository that still wants one of these lists it in its own `.tbhb/commits.toml`. The rules maintainer promotes a scope to the global list when `evaluate` reports show it means the same thing in three or more repositories.

Every scope that appears in one repository only goes to that repository's file, starting with the most used: `alpha`, `evals`, `designs`, `lint`, `lab`, `tony`, `decisions`, `o11y`, `adrs`, `content`, `skills`, and `tailnet`. Each repository's maintainer builds its file from its own `evaluate` reports, and that repository's usual review covers the file.

## Oddities

| Seen | Count | Outcome | What to write instead |
| --- | --- | --- | --- |
| `_` | 13 subjects | Fails the grammar, so no list can hold it. | No scope |
| `` `bazel` `` | not counted | Fails the grammar. | `bazel` |
| A path as scope | not counted | Fails the grammar, because `/` and `.` aren't allowed. | The scope of the component the path belongs to |
| An identifier such as `e-000001` | not counted | Matches the grammar but belongs on no list. | The scope of the area, with the identifier in the summary |
| `sync` as a type | 11 subjects | Not a type, pending D4. | `chore(sync)`, with `sync` in that repository's file |

## Types

The global file lists the 11 types `check_subject` allows today: `build`, `chore`, `ci`, `docs`, `feat`, `fix`, `perf`, `refactor`, `revert`, `style`, and `test`. Each carries the scope classes D3 recommends.
