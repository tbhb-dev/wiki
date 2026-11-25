---
type: Design Appendix
title: tbhbdev Go port evidence
description: Each claim the tbhbdev Go port design makes about tbhbdev, its neighbors, and the operator's rules, labeled existing, new, or unverified, with its source.
tags: [tbhbdev, go, evidence, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words gazelle golangci gobco gomutants govulncheck deadcode buildmeta userconfig agy ITSM gochecknoglobals -->

# tbhbdev Go port evidence

Back to [the design](README.md).

The gh evidence is in [appendix A](A-gh-architecture.md) and [appendix B](B-gh-pitfalls.md). This appendix covers everything else.

Labels:

- **existing** means a file read at the named commit.
- **outline** means the design outline from a tools read at `75cbfd83` on 2026-11-25, which this pass didn't read again.
- **brief** means the operator's rules as the task brief relays them.
- **new** means this design proposes it.
- **unverified** means nobody checked it.

## The operator's rules

| Claim | Label | Source |
| --- | --- | --- |
| tbhbdev ports to Go as a monorepo of plugin binaries, the Python pure-core refactor stops, and the first build is the front with a Python fallthrough | brief | the operator's decision of 2026-11-25 |
| The design follows gh after research into its pitfalls, the pure and thin I/O rule holds in Go, and no package takes the name `core` or `shell` | brief | the same |
| One commit builds one release bundle | brief | the same |
| Go has no `alpha`, commands get final names, and `alpha` works only through the fallthrough until each command ports | brief | the same |
| No version mismatch ever blocks a launch, and a host that can't run a driver's launch counts as a refusal | brief | the same |
| Go-only pull requests don't run the Python suite in pull requests or the merge queue | brief | the same |
| A separate CI speed role owns CI wall time across repositories | brief | the same |
| No decision record covered a Go port at the time of the outline's read, and the agent capacity record chose Go for new agent tooling | outline | grep of the decision folder |

## The tools repository

| Claim | Label | Source |
| --- | --- | --- |
| 37,745 source lines in `src/tbhbdev`, 95,572 test lines in 150 files, and 29,364 lines in `alpha` | outline | `wc -l` at `75cbfd83` |
| Per-module line counts in [appendix H](H-waves.md) | outline | `wc -l` at `75cbfd83` |
| `decisions/` is pure by its own docstring | outline | `decisions/__init__.py` at `75cbfd83` |
| `same_version` refuses a host whose `version.line()` differs | existing, outline | `src/tbhbdev/alpha/agent/decisions/driver.py` at `938f65f` line 69, and the outline's line 83 at `75cbfd83` |
| The forced command is `<agent home>/.local/bin/tbhbdev alpha agent ssh-serve` | outline | the agent README, line 41 |
| The envelope is `{"v": 1, "request": {"v": 2}}` | outline | the agent README, line 58, and `ssh_serve.py` line 89 |
| The launcher's exit codes are 0, 2, 3, 4, 124, and 143 | outline | the agent README |
| The launcher uses tmux and ssh, so Windows is out of scope | outline | `launcher.py` lines 15 to 29 |
| The `alpha` command list and usage text | existing | `src/tbhbdev/alpha/__init__.py` at `938f65f` |
| Top-level commands `admin`, `alpha`, `analytics`, `host`, and `secret` | existing | `src/tbhbdev/cli.py` at `938f65f` |
| `agy`, `adr`, `issue`, and `cue` are top-level tools | outline | the tools table at `75cbfd83` |
| Other packages add tools and config files through the `tbhbdev.plugins` entry point group | existing | `src/tbhbdev/plugins.py` at `938f65f` |
| graph and the private ITSM repository use that group | outline | their `pyproject.toml` files |
| `pyproject.toml` has one console script, `tbhbdev` | existing | `pyproject.toml` at `938f65f` |
| `pytest_test` takes `:repo_files` with a `long` timeout, near 300 seconds on CI | existing | `BUILD.bazel` at `938f65f` |
| The Python linters take `:repo_files` | existing | `BUILD.bazel` at `938f65f` |
| CI runs `bazelisk test --config=ci //...` | existing | `.github/workflows/ci.yml` at `938f65f` |
| CI gives the main cache token to `push` and `merge_group` runs | existing | the merge queue design's evidence, tools `ci.yml` at `f342d9c5` |
| Hosts install with `uv tool install git+https://github.com/tbhb-dev/tools` | outline | tools `AGENTS.md` |
| tools#385 (`da154050`) is the only verified open pure-core pull request | outline | local refs |
| `host` is frozen until ops replaces it | unverified | a memory note the outline cites |
| Which other repositories import the library-only modules | unverified | the tools README calls them libraries |
| Who writes host authorized-keys entries | unverified | |

## Neighbors and rules

| Claim | Label | Source |
| --- | --- | --- |
| awrel pins rules_go 0.65.0 and gazelle 0.54.0 and takes tbhb_rules_go as a development dependency | existing | awrel `MODULE.bazel` at `d677e5d` |
| afakt pins the same versions and uses `gazelle:prefix go.tbhb.dev/afakt` | outline | afakt `MODULE.bazel` and `BUILD.bazel` |
| awrel uses `go_arch_lint_test`, `gobco_test`, `gomutants_test`, `go_mod_tidy_test`, `go_deadcode_test`, and `govulncheck_test`, and `gazelle:prefix go.tbhb.dev/awrel` | existing | awrel `BUILD.bazel` at `d677e5d` |
| awrel stamps `Version`, `Commit`, and `Date` with `x_defs` from `STABLE_BUILD_SCM_*`, with a build-info fallback, and exempts them from `gochecknoglobals` | existing | awrel `internal/buildmeta` at `d677e5d` |
| tbhb_rules_go exports `go_arch_lint_test`, `golangci_lint_aspect_factory`, `golangci_lint_test_factory`, `gobco_test`, and `gomutants_test` | existing | rules `go/defs.bzl` at `fc0074a` |
| rules has no ast-grep rule set | outline, existing | the outline's grep, and a grep of rules at `fc0074a` |
| The size gate counts 400 code and 1,500 raw added lines, with Markdown exempt and `go.sum` exempt | existing | rules `lint/private/pr_size.sh.tpl` and `pr-size-exempt.txt` at `fc0074a` |
| The rules repository checks run the size gate on every pull request | existing | rules `.github/workflows/repo-checks.yml` at `fc0074a` |
| Neither afakt nor awrel CI signs or attests releases | outline | grep for cosign and attest |
| afakt and awrel build releases with `bazelisk build --config=release` | outline | their CI files |
| The wiki proves a lint rule with `expect_failure_test` | existing | wiki `BUILD.bazel` on main |

## What this design adds

Everything in [appendix E](E-plugin-contract.md), [appendix F](F-build-and-checks.md), and [appendix G](G-names-and-migration.md) that names no existing file is new: the front, the contract, the manifest, the bundle and installer, the forwarder, `names.toml`, the envelope corpus, the test macros, and the ast-grep rules.
