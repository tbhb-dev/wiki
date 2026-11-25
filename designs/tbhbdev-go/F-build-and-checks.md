---
type: Design Appendix
title: tbhbdev Go port build and checks
description: The Bazel layout of the tbhbdev Go port beside the Python, the disjoint inputs that keep the Python suite off Go-only changes, and each check that runs from the first commit.
tags: [tbhbdev, go, bazel, checks, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words selfupdate flock gazelle depguard golangci gobco gomutants govulncheck deadcode genquery buildinfo termio procexec statedir ghapi sshcall gochecknoglobals httptest gostd sgconfig deptry strconv lookpath -->

# tbhbdev Go port build and checks

Back to [the design](README.md).

## Layout

```text
tools/
  MODULE.bazel        rules_go 0.65.0, gazelle 0.54.0, tbhb_rules_go, as awrel pins them
  go.mod              module go.tbhb.dev/tools, no go.work
  BUILD.bazel         gazelle, the whole-module Go checks, repository hygiene
  .golangci.yml       linters, depguard lists, gochecknoglobals
  .go-arch-lint.yml   components and allowed imports
  sgconfig.yml        ast-grep rule directories
  cmd/tbhbdev/                  the front, one-line main
  cmd/tbhbdev-<name>/           one plugin each, one-line main
  cmd/tbhbdev-self-update/      the self-update helper, one-line main
  internal/<domain>/<topic>/    pure packages
  internal/<domain>/io/<name>/  I/O adapters, and io/app for wiring
  bundle/                       the release archive, manifest, and main.json pointer targets
  schemas/agent-protocol.json   the protocol schema, with its decisions file and frozen versions
  src/tbhbdev/                  Python, as today
  tests/                        Python tests, with pytest_test moved here
  tests/agent-corpus/           request envelopes both languages test against
```

gazelle writes one `BUILD.bazel` per Go package with `# gazelle:prefix go.tbhb.dev/tools` at the root, as afakt and awrel do with their own prefixes. There's no plain `go build` path, and release builds use `bazelisk build --config=release` as in afakt and awrel CI.

## Disjoint inputs

Today `//:pytest_test` in tools takes `srcs = [":repo_files"]`, the whole tree, and has a `long` timeout because the suite runs near 300 seconds on CI runners (existing). Any change, Go included, makes it run again.

The port splits the inputs (D7):

| Target | Package | Inputs |
| --- | --- | --- |
| `pytest_test` | `//tests` | `src/tbhbdev/**`, `tests/**` except the corpus, `pyproject.toml`, `uv.lock`, and the pytest config |
| ruff, vulture, complexipy, bandit, and deptry tests | `//tests` or `//src/tbhbdev` | The Python tree only |
| Each `go_test` | its own package | That package's sources and its Go dependencies |
| Whole-module Go checks | root | `go.mod`, `go.sum`, the lint configs, and the Go packages |
| Corpus tests | `//tests/agent-corpus` | The corpus, the protocol schema, plus the Python server or the Go server under test |
| Repository hygiene: gitleaks, REUSE, editorconfig, and formatting | root | The whole tree, as today |

Two `genquery` targets list the source files under `//tests:pytest_test` and under every Go test, and a shell test fails if the first holds a `.go` file or the second holds a file under `src/tbhbdev`. No target named `all`, `everything`, or similar collects both languages.

With the inputs apart, a Go-only change leaves `pytest_test` with unchanged inputs. Bazel then reuses its result wherever the remote cache serves the run. tools CI gives the main cache token to `push` and `merge_group` runs (existing), so merge queue runs read main's results. Whether pull request runs read them is unverified, and the CI speed role decides whether to add affected-target selection there.

## Checks

| Check | Rule | Source |
| --- | --- | --- |
| Pull request size | 400 added code lines and 1,500 raw lines, Go tests included, Markdown exempt | existing: rules `lint/private/pr_size.sh.tpl`, run by the rules `repo-checks.yml` |
| Import direction | `go_arch_lint_test` with the components below | existing macro in rules `go/defs.bzl`, new config |
| Standard-library bans | depguard through the golangci-lint aspect | existing factory in rules `go/defs.bzl`, new lists |
| No package globals | `gochecknoglobals`, with one exception in `internal/buildinfo` | existing linter, as in awrel's `buildmeta` |
| I/O in pure code and unit tests | `ast_grep_test` | new macro in rules, plan step 1 |
| Test time caps | `unit_go_test` and `io_go_test` | new macros in tools |
| The front budget | Imports and size of `cmd/tbhbdev` | new |
| Every binary stamped | Each bundle binary prints the bundle version for `--version` | new release test |
| Contract | One test per contract variable, and no plugin exit code in 125 to 127 | new |
| Protocol version | The `protocol-version` CI job compares the schema at the merge base with the head and requires a matching decision entry ([appendix I](I-protocol.md#the-schema-version-check)) | new, a tools CI job outside Bazel, since it needs the merge base |
| Protocol corpus | Writers emit only schema-valid documents, both servers read each served version, and the Python and Go schema copies match byte for byte | new |
| Self-update | `unit_go_test` tables for `selfupdate.Decide`, and an `io_go_test` that runs two swaps at once, prunes beside a held bundle, and feeds each failure in [appendix J](J-self-update.md#failures) | new |
| Coverage and mutation | `gobco_test` at 100% branch coverage on pure packages, `gomutants_test` on changed functions | existing macros, as awrel runs them |
| Module hygiene | `go_mod_tidy_test`, `go_deadcode_test`, `govulncheck_test` | existing macros, as awrel runs them |

Each new rule ships with a planted fixture that must fail it, run through `expect_failure_test` as the wiki's IP address proof does.

### Components

| Component | Paths | May import |
| --- | --- | --- |
| front | `cmd/tbhbdev` | `internal/dispatch/io/front` |
| front wiring | `internal/dispatch/io/front` | `internal/dispatch`, `internal/buildinfo`, `internal/plugin`, `internal/selfupdate` |
| plugin mains | `cmd/tbhbdev-*` | `internal/plugin/io/run`, the plugin's own `io/app` |
| io | `internal/**/io/**` | io and pure packages |
| pure | the rest of `internal/**` | pure packages only |

A domain's packages import another domain's only through a package that the arch config names as shared. That stops the cross-command imports gh grew ([appendix B](B-gh-pitfalls.md#26-shared-code-for-checks-that-diverged)).

go-arch-lint may not police the standard library (unverified), so depguard does. In pure files it allows only `bytes`, `cmp`, `encoding/json`, `errors`, `fmt`, `maps`, `math`, `path`, `path/filepath`, `regexp`, `slices`, `sort`, `strconv`, `strings`, `time`, `unicode`, `unicode/utf8`, and the module's pure packages. ast-grep then bans the calls in `time` and `path/filepath` that reach the clock or the disk.

### ast-grep rules

The rules live in tbhb-dev/rules for other Go repositories to use, behind an `ast_grep_test` macro (new: no ast-grep rule set exists in rules today, per the outline's search).

| Rule | Where | Forbids |
| --- | --- | --- |
| `pure-no-io` | pure sources | `os.$F(...)`, `exec.$F(...)`, `net.$F(...)`, `http.$F(...)`, `time.Now()`, `time.Sleep(...)`, `time.After(...)`, `filepath.Glob/Walk/WalkDir/Abs/EvalSymlinks(...)` |
| `unit-test-no-io` | `_test.go` in pure packages | All of the above, plus `t.TempDir()`, `t.Setenv(...)`, `httptest.$F(...)`, and `os.Getenv(...)` |
| `front-no-lookpath` | `internal/dispatch/**` | `exec.LookPath(...)` and `os.Getenv("PATH")` |
| `front-no-net` | `internal/dispatch/**` and `cmd/tbhbdev` | `net.$F(...)`, `http.$F(...)`, and any import of `internal/selfupdate/io/**` |
| `one-http-client` | everywhere except `internal/ghapi/io/**` | `http.Client{...}` and `&http.Client{...}` |
| `no-alpha` | all Go sources except `internal/dispatch` | the string literal `"alpha"`, so no Go code names an `alpha` command (D5) |

### Test tiers and time caps

gazelle's `map_kind` directive makes every generated `go_test` one of two kinds (D13):

- **`unit_go_test`**, in pure packages: `size = "small"` and `args = ["-test.timeout=10s"]`, so a unit test target fails past 10 seconds. These tests take plain values and use no fakes.
- **`io_go_test`**, under `io/`: `size = "medium"`, so Bazel stops it at 300 seconds. These tests run real tools in a temporary root with a scrubbed environment from one helper, and match fakes by request content.

A `genquery` test fails on any `go_test` in the module that isn't one of those kinds, or that sets its own `timeout`. With the targets in parallel, the Go suite's wall time stays under 5 minutes, and the CI speed role tracks it across repositories.

### The front budget

- `go_arch_lint_test` keeps the front's imports to the four packages in the components table and the standard library.
- A size test fails if the stripped front binary grows past a budget set from the first build (new, value to be measured).
- A test runs the front under a process counter and fails if it starts any process before its exec. The front's only reads are the manifest and the self-update cache, plus the shared lock on its bundle.
