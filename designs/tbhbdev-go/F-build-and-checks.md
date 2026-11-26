---
type: Design Appendix
title: tbhbdev Go port build and checks
description: The Bazel layout of the tbhbdev Go port in the tbhb-dev/tbhbdev monorepo, how the Go and Python build inputs stay apart, and each check that runs from the first commit.
tags: [tbhbdev, go, bazel, checks, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words selfupdate flock gazelle depguard golangci gobco gomutants govulncheck deadcode genquery buildinfo termio procexec statedir ghapi sshcall gochecknoglobals httptest gostd sgconfig deptry strconv lookpath cobra koanf cuelang gochecknoinits gomodguard pflag stdlib ghinstallation githubv -->

# tbhbdev Go port build and checks

Back to [the design](README.md).

## Layout

The Go port lives in the monorepo tbhb-dev/tbhbdev (D22). The monorepo session owns its root and its general scaffolding. The CLI's tree sits under `factory/cli`, the folder tbhb chose (D27).

```text
tbhb-dev/tbhbdev/
  MODULE.bazel        rules_go 0.65.0, gazelle 0.54.0, tbhb_rules_go, as awrel pins them
  go.mod              declares module github.com/tbhb-dev/tbhbdev, no go.work
  BUILD.bazel         gazelle, the whole-module Go checks, repository hygiene
  .golangci.yml       linters, depguard lists, gomodguard bans, gochecknoglobals, gochecknoinits
  .go-arch-lint.yml   components, allowed imports, and vendor lists, with paths under factory/cli
  sgconfig.yml        ast-grep rule directories
  factory/cli/
    cmd/tbhbdev/                  the Go binary, one-line main
    internal/<domain>/<topic>/    pure packages
    internal/<domain>/io/<name>/  I/O adapters, and io/app for wiring and cobra commands
    internal/schemas/cue/         byte-checked copies of the local CUE schemas (D20)
    internal/agent/protocol/      a byte-checked copy of the local protocol schema
    bundle/                       the release archive, manifest, and main.json pointer targets
    tests/agent-corpus/           request envelopes both languages test against
```

The CLI import brings the Python sources, tests, and schemas under `factory/cli`. The `protocol-version` job moves to monorepo CI.

gazelle writes one `BUILD.bazel` per Go package. The root omits the `# gazelle:prefix` directive because gazelle v0.54.0 reads the module path from `go.mod` when the prefix is unset ([source](https://github.com/bazel-contrib/bazel-gazelle/blob/v0.54.0/language/go/config.go#L747-L760)). Bazel labels name the folder as `//factory/cli/...`. There is no plain `go build` path, and release builds use `bazelisk build --config=release` as in afakt and awrel CI.

## Build inputs

Today `//:pytest_test` in tools takes `srcs = [":repo_files"]`, the whole tree, and has a `long` timeout because the suite runs near 300 seconds on CI runners (existing). Any change makes it run again.

After import, separate inputs keep a Go-only change from starting the Python suite (D7).

| Target | Repository and package | Inputs |
| --- | --- | --- |
| `pytest_test` and the Python linters | tbhbdev, under `factory/cli` | The imported Python tree |
| Each `go_test` | tbhbdev, its own package under `factory/cli` | That package's sources and its Go dependencies |
| Whole-module Go checks | tbhbdev root | `go.mod`, `go.sum`, the lint configs, and the Go packages |
| Schema copy checks | tbhbdev, `//factory/cli/internal/schemas` and `//factory/cli/internal/agent/protocol` | The embedded copies and their local source files |
| Corpus tests | tbhbdev, `//factory/cli/tests/agent-corpus` | The corpus, the protocol schema, and the Go server or the locally built Python wheel under test |
| Repository hygiene: gitleaks, REUSE, editorconfig, and formatting | tbhbdev root | The whole tree |

The copy checks and corpus tests use local Bazel inputs from `factory/cli`. No target named `all`, `everything`, or similar collects both languages.

## Checks

| Check | Rule | Source |
| --- | --- | --- |
| Pull request size | 400 added code lines and 1,500 raw lines, Go tests included, Markdown exempt | existing: rules `lint/private/pr_size.sh.tpl`, run by the rules `repo-checks.yml` |
| tbhb gate paths | A job labels a pull request `tbhb-gate` when it changes one of the two gated paths in D19, and only then | new, a monorepo CI job outside Bazel, since it reads the pull request's changed paths |
| Import direction | `go_arch_lint_test` with the components below | existing macro in rules `go/defs.bzl`, new config |
| Standard-library bans | depguard through the golangci-lint aspect | existing factory in rules `go/defs.bzl`, new lists |
| No package globals | `gochecknoglobals`, with one exception in `internal/buildinfo` | existing linter, as in awrel's `buildmeta` |
| The Go stack in its place | cobra, pflag, koanf, `cuelang.org/go`, and go-github only in their packages, no `init()`, no viper, `githubv4`, or `ghinstallation`, no `os/exec` outside one package, and no `cue` child process by name or path ([appendix K](K-go-stack.md#the-rules-that-hold-it)) | existing linters and go-arch-lint vendor lists, as awrel uses them, with new config |
| The CUE boundary | `run.Main` clears the `CUE_` variables before any CUE call, code reachable from validation reads no other variable, the environment reads in linked `pkg/tool` task code stay unreachable, and no task runner is linked ([appendix K](K-go-stack.md#the-cue-boundary)) | new |
| input/output (I/O) in pure code and unit tests | `ast_grep_test` | new macro in rules, plan step 1 |
| Test time caps | `unit_go_test` and `io_go_test` | new macros in the monorepo |
| The front budget | Imports and startup work in dispatch | new |
| Build stamp | `tbhbdev version` prints the bundle version, and both it and `tbhbdev-py --version` report the bundle's monorepo commit | new release test |
| Protocol version | The `protocol-version` CI job compares the schema at the merge base with the head and requires a matching decision entry ([appendix I](I-protocol.md#the-schema-version-check)) | new, outside Bazel because it needs the merge base |
| Protocol corpus | Writers emit only schema-valid documents, both servers read each served version and the legacy shape, the staged request file matches the envelope's request, and the packaged and embedded schema copies match byte for byte | new |
| Self-update | The tests that [appendix J](J-self-update.md#tests) lists | new |
| Coverage and mutation | `gobco_test` at 100% branch coverage on pure packages, `gomutants_test` on changed functions | existing macros, as awrel runs them |
| Module hygiene | `go_mod_tidy_test`, `go_deadcode_test`, `govulncheck_test` | existing macros, as awrel runs them |

Each new rule ships with a planted fixture that must fail it, run through `expect_failure_test` as the wiki's IP address proof does.

### Components

| Component | Paths | May import |
| --- | --- | --- |
| main | `factory/cli/cmd/tbhbdev` | `factory/cli/internal/dispatch/io/front`, command wiring under `io/app`, `factory/cli/internal/selfupdate/io/swap` |
| front wiring | `factory/cli/internal/dispatch/io/front` | `factory/cli/internal/dispatch`, `factory/cli/internal/buildinfo`, `factory/cli/internal/selfupdate` |
| io | `factory/cli/internal/**/io/**` | io and pure packages |
| pure | the rest of `factory/cli/internal/**` | pure packages only |

A domain's packages import another domain's only through a package that the arch config names as shared. That stops the cross-command imports gh grew ([appendix B](B-gh-pitfalls.md#26-shared-code-for-checks-that-diverged)).

depguard checks standard-library imports. In pure files it allows only `bytes`, `cmp`, `embed`, `encoding/json`, `errors`, `fmt`, `maps`, `math`, `path`, `path/filepath`, `regexp`, `slices`, `sort`, `strconv`, `strings`, `time`, `unicode`, `unicode/utf8`, and the module's pure packages. The two schema packages may also import the `cuelang.org/go` packages that [appendix K](K-go-stack.md#the-libraries) lists. ast-grep then bans the calls in `time` and `path/filepath` that reach the clock or the disk.

### ast-grep rules

The rules live in tbhb-dev/rules for other Go repositories to use, behind an `ast_grep_test` macro (new: no ast-grep rule set exists in rules today, per the outline's search).

| Rule | Where | Forbids |
| --- | --- | --- |
| `pure-no-io` | pure sources | `os.$F(...)`, `exec.$F(...)`, `net.$F(...)`, `http.$F(...)`, `time.Now()`, `time.Sleep(...)`, `time.After(...)`, `filepath.Glob/Walk/WalkDir/Abs/EvalSymlinks(...)` |
| `unit-test-no-io` | `_test.go` in pure packages | All of the above, plus `t.TempDir()`, `t.Setenv(...)`, `httptest.$F(...)`, and `os.Getenv(...)` |
| `front-no-lookpath` | `factory/cli/internal/dispatch/**` | `exec.LookPath(...)` and `os.Getenv("PATH")` |
| `front-no-net` | `factory/cli/internal/dispatch/**` | `net.$F(...)`, `http.$F(...)`, and any import of `factory/cli/internal/selfupdate/io/**` |
| `one-http-client` | everywhere except `factory/cli/internal/ghapi/io/**` | `http.Client{...}` and `&http.Client{...}` |
| `no-alpha` | all Go sources except `factory/cli/internal/dispatch` | the string literal `"alpha"`, so no Go code names an `alpha` command (D5) |

Three more rules, `no-cue-binary`, `cue-one-context`, and `no-cobra-globals`, stay in the monorepo's own `sgconfig.yml`, since they hold this module's stack ([appendix K](K-go-stack.md#the-rules-that-hold-it)).

### Test tiers and time caps

gazelle's `map_kind` directive makes every generated `go_test` one of two kinds (D13):

- **`unit_go_test`**, in pure packages: `size = "small"` and `args = ["-test.timeout=10s"]`, so a unit test target fails past 10 seconds. These tests take plain values and use no fakes.
- **`io_go_test`**, under `io/`: `size = "medium"`, so Bazel stops it at 300 seconds. These tests run real tools in a temporary root with a scrubbed environment from one helper, and match fakes by request content.

A `genquery` test fails on any `go_test` in the module that is not one of those kinds, or that sets its own `timeout`. With the targets in parallel, the Go suite's wall time stays under 5 minutes, and tbhb-agent tracks it across repositories as separate CI speed work.

### The front budget

- `go_arch_lint_test` keeps dispatch wiring to the packages in the components table, plus the standard library. depguard's `front-stdlib-only` list holds the same line for third-party packages, and a planted fixture that imports cobra into the dispatch wiring fails it.
- A test runs the front under a process counter and fails if it starts any process during routing. With a shared lock on its bundle, the front reads the manifest and the self-update cache.
