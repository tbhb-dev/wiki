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

<!-- cspell:words gazelle golangci gobco gomutants govulncheck deadcode buildmeta userconfig agy ITSM gochecknoglobals cobra pflag koanf viper cuelang jsonschema gomodguard depguard -->

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
| The design follows gh after research into its pitfalls, the pure and thin input/output (I/O) rule holds in Go, and no package takes the name `core` or `shell` | brief | the same |
| One commit builds one release bundle | brief | the same |
| Go has no `alpha`, commands get final names, and `alpha` works only through the fallthrough until each command ports | brief | the same |
| No version mismatch ever blocks a launch, and a host that can't run a driver's launch counts as a refusal | brief | the same |
| Driver and host check a protocol version and never build hashes, and a build mismatch is a logged warning | brief | the operator's requirements of 2026-11-25, relayed after the first draft |
| tbhbdev updates itself at start when behind main, with a cached main commit, an atomic swap, no refusal on failure, and an opt-out variable | brief | the same |
| The run request and run metadata are one versioned schema that both sides validate against, with a continuous integration (CI) check on version decisions | brief | the same |
| The tools maintainer is building event-driven sync on merges to main | brief, unverified | the same. Its design wasn't read |
| Go-only pull requests don't run the Python suite in pull requests or the merge queue | brief | the same |
| A separate CI speed role owns CI wall time across repositories | brief | the same |
| No decision record covered a Go port at the time of the outline's read, and the agent capacity record chose Go for new agent tooling | outline | grep of the decision folder |
| Wave 1 is the unfinished review tooling, the launcher moves to a later wave, protocol versioning and self-update stay, and implementation waits for the operator's read of the merged design | brief | the operator's decisions of 2026-11-25 on wave order and stack (D17) |
| Plugins use cobra, pflag, and koanf, with explicit dependencies, no package state, and a standard-library front | brief | the same (D18) |
| Scaffolding pull requests go to the operator before merge | brief | the same (D19) |
| Go validates schemas in-process with `cuelang.org/go`, embedded by `go:embed`, and never runs the `cue` binary | brief | the same (D20) |
| After tools#399, Python gets no new review features | brief | the same relay |

## The tools repository

| Claim | Label | Source |
| --- | --- | --- |
| 37,745 source lines in `src/tbhbdev`, 95,572 test lines in 150 files, and 29,364 lines in `alpha` | outline | `wc -l` at `75cbfd83` |
| Per-module line counts in [appendix H](H-waves.md) | outline | `wc -l` at `75cbfd83` |
| `decisions/` is pure by its own docstring | outline | `decisions/__init__.py` at `75cbfd83` |
| `same_version` refuses a host whose `version.line()` differs | existing | `src/tbhbdev/alpha/agent/decisions/driver.py` line 83 at `bf420ff`, called from `driver.py` line 374 |
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
| `jsonschema` is a tools runtime dependency | existing | `pyproject.toml` at `bf420ff` |
| `pytest_test` takes `:repo_files` with a `long` timeout, near 300 seconds on CI | existing | `BUILD.bazel` at `938f65f` |
| The Python linters take `:repo_files` | existing | `BUILD.bazel` at `938f65f` |
| CI runs `bazelisk test --config=ci //...` | existing | `.github/workflows/ci.yml` at `938f65f` |
| CI gives the main cache token to `push` and `merge_group` runs | existing | the merge queue design's evidence, tools `ci.yml` at `f342d9c5` |
| Hosts install with `uv tool install git+https://github.com/tbhb-dev/tools` | outline | tools `AGENTS.md` |
| tools#385 (`da154050`) is the only verified open pure-core pull request | outline | local refs. The tools inventory of 2026-11-25 lists it as merged since |
| `host` is frozen until ops replaces it | unverified | a memory note the outline cites |
| Which other repositories import the library-only modules | unverified | the tools README calls them libraries |
| Who writes host authorized-keys entries | unverified | |

## The agent protocol today

Read at tools `bf420ff`, main on 2026-11-25.

| Claim | Label | Source |
| --- | --- | --- |
| The envelope must have exactly `v`, `request`, and `push_token`, and `v` must equal 1 | existing | `src/tbhbdev/alpha/agent/ssh_serve.py` lines 85 to 89 |
| The request's `v` must equal `REQUEST_VERSION`, which is 2, and any unknown field is refused | existing | `decisions/request.py` lines 282 to 295, and `model.py` line 16 |
| No schema file, protocol version, or negotiation exists in `alpha/agent`, and the run directory files carry no version | existing | `git grep` for `schema`, `protocol`, and `version` under `src/tbhbdev/alpha/agent`, and `decisions/runfiles_names.py` |
| Preflight prints the host's `tbhbdev --version` and exits early when it differs from the driver's | existing | `decisions/remote.py` lines 163 to 175 |
| `run_host.py` and `decisions/argv.py` took 47 commits from 2026-11-01 to 2026-11-25 | existing, brief | `git log --since=2026-11-01 main -- <both files>` |

The brief says tools main has no protocol or schema version in `alpha/agent`. The two integer checks above do exist, so this design reads the brief as no schema file and no negotiated version, which the read confirms.

## The review tooling

The merged and open lists in [appendix H](H-waves.md#wave-1-the-review-tooling) come from the tools inventory of 2026-11-25, which read `git log origin/main` and the `src/tbhbdev` tree of tools, and two coordination decision lines it names. This pass relays it and checked the rows below against the local tools mirror at `bf420ff`, which predates several of the listed pull requests.

| Claim | Label | Source |
| --- | --- | --- |
| The merged review pull requests and their files, and tools#399 open, approved, and queued | brief | the tools inventory of 2026-11-25 |
| The unbuilt pieces: the M4 proof, the M5 merge-gate design, the paired launch and summary, and the review debt | brief, unverified | the inventory, from the milestone plan, not checked against a tracker |
| tools#358 and tools#365 are unmerged | brief, unverified | the inventory, from the git log only |
| `review` 309, `review_doc` 427, `review_policy` 242, `review_prompt` 121, `agent/review_post.py` 177, `content_vet` 186, `gh_app` 815, and `cue_runtime` 156 lines | existing | `wc -l` at `bf420ff` |
| `alpha review` has the subcommands `draft`, `check`, `policy`, `post`, `dismiss`, and `reply` | existing | `src/tbhbdev/alpha/review.py` lines 275 to 285 at `bf420ff` |
| `review_policy.py` validates with `jsonschema` 2020-12, and the review draft schema is JSON Schema in `review_data/` | existing | `review_policy.py` line 12 and `git ls-tree` at `bf420ff` |
| A review run's verdict comes back as text, and the run host posts it with a run-scoped token | existing | `src/tbhbdev/alpha/agent/credentials.py` lines 38 to 40 and `review_post.py` line 1 at `bf420ff` |

## The Go stack

| Claim | Label | Source |
| --- | --- | --- |
| `src/tbhbdev/schemas/core.cue`, `issue.cue`, and `review.cue` are symbolic links to the root `schemas/`, which also holds `taxonomy.cue` | existing | `git ls-tree -l` at `bf420ff` |
| `content_vet.py` runs `cue vet` and `cue export` through the binary that `cue_runtime.py` downloads and pins | existing | `content_vet.py` lines 140 to 186 and `cue_runtime.py` lines 1 to 8 at `bf420ff` |
| `userconfig` reads `$XDG_CONFIG_HOME/tbhbdev/config.toml` on first need, with no defaults and no environment overrides | existing | the `userconfig.py` docstring at `bf420ff` |
| awrel builds its root command with cobra 1.10.2 from a function that takes its streams and environment lookup | existing | awrel `go.mod` and `cmd/awrel/main.go` at `d677e5d` |
| awrel's go-arch-lint config sets `depOnAnyVendor: false` and lists cobra under `vendors` and `canUse` | existing | awrel `.go-arch-lint.yml` at `d677e5d` |
| awrel bans modules with gomodguard_v2 and packages with depguard | existing | awrel `.golangci.yml` at `d677e5d` |
| Go's `embed` skips symbolic links | unverified | Go's `embed` documentation |
| `cuelang.org/go/encoding/jsonschema` reads JSON Schema documents for in-process validation | unverified | the `cuelang.org/go` documentation, checked by the schema slice |
| cobra answers a hidden `__complete` command | unverified | cobra's documentation |
| viper's usual entry points act on a package-level default instance, and koanf has none | unverified | each library's documentation |

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

Everything in appendices [E](E-plugin-contract.md), [F](F-build-and-checks.md), [G](G-names-and-migration.md), [I](I-protocol.md), [J](J-self-update.md), and [K](K-go-stack.md) that names no existing file is new. That covers the front, the contract, the manifest, the bundle and installer, the forwarder, `names.toml`, the envelope corpus, the test macros, and the ast-grep rules. It also covers the protocol schema, its decisions file and CI job, the release pointer, self-update, the cobra wiring, the koanf loader, the schema package, and the stack rules.
