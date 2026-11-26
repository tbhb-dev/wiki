---
type: Design Appendix
title: tbhbdev Go port build and checks
description: The Bazel layout of the tbhbdev Go port in the tbhb-dev/tbhbdev monorepo, how its inputs stay apart from the Python in tools, and each check that runs from the first commit.
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
  MODULE.bazel        rules_go 0.65.0, gazelle 0.54.0, tbhb_rules_go, as awrel pins them, and the pinned tools commit
  go.mod              module github.com/tbhb-dev/tbhbdev, the only place the path is written, no go.work
  BUILD.bazel         gazelle, the whole-module Go checks, repository hygiene
  .golangci.yml       linters, depguard lists, gomodguard bans, gochecknoglobals, gochecknoinits
  .go-arch-lint.yml   components, allowed imports, and vendor lists, with paths under factory/cli
  sgconfig.yml        ast-grep rule directories
  factory/cli/
    cmd/tbhbdev/                  the front, one-line main
    cmd/tbhbdev-<name>/           one plugin each, one-line main
    cmd/tbhbdev-self-update/      the self-update helper, one-line main
    internal/<domain>/<topic>/    pure packages
    internal/<domain>/io/<name>/  I/O adapters, and io/app for wiring and cobra commands
    internal/schemas/cue/         byte-checked copies of the tools CUE schemas (D20)
    internal/agent/protocol/      a byte-checked copy of the tools protocol schema
    bundle/                       the release archive, manifest, and main.json pointer targets
    tests/agent-corpus/           request envelopes both languages test against
```

tbhb-dev/tools keeps `src/tbhbdev/`, its Python tests, `schemas/agent-protocol.json` with its decisions file and frozen versions, and the `protocol-version` job.

gazelle writes one `BUILD.bazel` per Go package. The root carries no `# gazelle:prefix` directive, since gazelle takes the prefix from `go.mod` (unverified against gazelle 0.54.0). If it needs the directive, a test fails when the directive and `go.mod` differ, so a rename still edits one value by hand. Bazel labels name the folder as `//factory/cli/...`. There is no plain `go build` path, and release builds use `bazelisk build --config=release` as in afakt and awrel CI.

## Inputs across the two repositories

Today `//:pytest_test` in tools takes `srcs = [":repo_files"]`, the whole tree, and has a `long` timeout because the suite runs near 300 seconds on CI runners (existing). Any change makes it run again.

With the Go code in its own repository, a Go change never starts the tools suite (D7). Inside each repository, inputs stay apart:

| Target | Repository and package | Inputs |
| --- | --- | --- |
| `pytest_test` and the Python linters | tools, as today | The Python tree. Splitting `:repo_files` is separate CI speed work for tbhb-agent, and no longer part of this port |
| Each `go_test` | tbhbdev, its own package under `factory/cli` | That package's sources and its Go dependencies |
| Whole-module Go checks | tbhbdev root | `go.mod`, `go.sum`, the lint configs, and the Go packages |
| Schema copy checks | tbhbdev, `//factory/cli/internal/schemas` and `//factory/cli/internal/agent/protocol` | The embedded copies and the same files from the pinned tools commit |
| Corpus tests | tbhbdev, `//factory/cli/tests/agent-corpus` | The corpus, the protocol schema, and the Go server or the pinned tools wheel under test |
| Repository hygiene: gitleaks, REUSE, editorconfig, and formatting | each repository's root | The whole tree, as today |

The pinned tools commit is a Bazel repository in `MODULE.bazel`, so the copy checks and corpus tests run again only on a pin bump or a change to their Go side. No target named `all`, `everything`, or similar collects both languages.

## Checks

| Check | Rule | Source |
| --- | --- | --- |
| Pull request size | 400 added code lines and 1,500 raw lines, Go tests included, Markdown exempt | existing: rules `lint/private/pr_size.sh.tpl`, run by the rules `repo-checks.yml` |
| tbhb gate paths | A job labels a pull request `tbhb-gate` when it changes one of the three gated paths in D19, and only then | new, a monorepo CI job outside Bazel, since it reads the pull request's changed paths |
| Import direction | `go_arch_lint_test` with the components below | existing macro in rules `go/defs.bzl`, new config |
| Standard-library bans | depguard through the golangci-lint aspect | existing factory in rules `go/defs.bzl`, new lists |
| No package globals | `gochecknoglobals`, with one exception in `internal/buildinfo` | existing linter, as in awrel's `buildmeta` |
| The Go stack in its place | cobra, pflag, koanf, `cuelang.org/go`, and go-github only in their packages, no `init()`, no viper, `githubv4`, or `ghinstallation`, no `os/exec` outside one package, and no `cue` child process by name or path ([appendix K](K-go-stack.md#the-rules-that-hold-it)) | existing linters and go-arch-lint vendor lists, as awrel uses them, with new config |
| The CUE boundary | `run.Main` clears the `CUE_` variables before any CUE call, code reachable from validation reads no other variable, the environment reads in linked `pkg/tool` task code stay unreachable, and no task runner is linked ([appendix K](K-go-stack.md#the-cue-boundary)) | new |
| input/output (I/O) in pure code and unit tests | `ast_grep_test` | new macro in rules, plan step 1 |
| Test time caps | `unit_go_test` and `io_go_test` | new macros in the monorepo |
| The front budget | Imports and size of `factory/cli/cmd/tbhbdev` | new |
| Every binary stamped | Each bundle binary prints the bundle version for `--version` | new release test |
| Contract | One test per contract variable, and no plugin exit code in 125 to 127 | new |
| Protocol version | The `protocol-version` CI job compares the schema at the merge base with the head and requires a matching decision entry ([appendix I](I-protocol.md#the-schema-version-check)) | new, a tools CI job outside Bazel, since it needs the merge base |
| Protocol corpus | Writers emit only schema-valid documents, both servers read each served version and the legacy shape, the staged request file matches the envelope's request, and the tools and monorepo schema copies match byte for byte | new |
| Self-update | The tests that [appendix J](J-self-update.md#tests) lists | new |
| Coverage and mutation | `gobco_test` at 100% branch coverage on pure packages, `gomutants_test` on changed functions | existing macros, as awrel runs them |
| Module hygiene | `go_mod_tidy_test`, `go_deadcode_test`, `govulncheck_test` | existing macros, as awrel runs them |

Each new rule ships with a planted fixture that must fail it, run through `expect_failure_test` as the wiki's IP address proof does.

### Components

| Component | Paths | May import |
| --- | --- | --- |
| front | `factory/cli/cmd/tbhbdev` | `factory/cli/internal/dispatch/io/front` |
| front wiring | `factory/cli/internal/dispatch/io/front` | `factory/cli/internal/dispatch`, `factory/cli/internal/buildinfo`, `factory/cli/internal/plugin`, `factory/cli/internal/selfupdate` |
| plugin mains | `factory/cli/cmd/tbhbdev-*`, except the self-update helper | `factory/cli/internal/plugin/io/run`, the plugin's own `io/app` |
| self-update helper main | `factory/cli/cmd/tbhbdev-self-update` | `factory/cli/internal/selfupdate/io/swap` |
| io | `factory/cli/internal/**/io/**` | io and pure packages |
| pure | the rest of `factory/cli/internal/**` | pure packages only |

A domain's packages import another domain's only through a package that the arch config names as shared. That stops the cross-command imports gh grew ([appendix B](B-gh-pitfalls.md#26-shared-code-for-checks-that-diverged)).

go-arch-lint may not police the standard library (unverified), so depguard does. In pure files it allows only `bytes`, `cmp`, `embed`, `encoding/json`, `errors`, `fmt`, `maps`, `math`, `path`, `path/filepath`, `regexp`, `slices`, `sort`, `strconv`, `strings`, `time`, `unicode`, `unicode/utf8`, and the module's pure packages. The two schema packages may also import the `cuelang.org/go` packages that [appendix K](K-go-stack.md#the-libraries) lists. ast-grep then bans the calls in `time` and `path/filepath` that reach the clock or the disk.

### ast-grep rules

The rules live in tbhb-dev/rules for other Go repositories to use, behind an `ast_grep_test` macro (new: no ast-grep rule set exists in rules today, per the outline's search).

| Rule | Where | Forbids |
| --- | --- | --- |
| `pure-no-io` | pure sources | `os.$F(...)`, `exec.$F(...)`, `net.$F(...)`, `http.$F(...)`, `time.Now()`, `time.Sleep(...)`, `time.After(...)`, `filepath.Glob/Walk/WalkDir/Abs/EvalSymlinks(...)` |
| `unit-test-no-io` | `_test.go` in pure packages | All of the above, plus `t.TempDir()`, `t.Setenv(...)`, `httptest.$F(...)`, and `os.Getenv(...)` |
| `front-no-lookpath` | `factory/cli/internal/dispatch/**` | `exec.LookPath(...)` and `os.Getenv("PATH")` |
| `front-no-net` | `factory/cli/internal/dispatch/**` and `factory/cli/cmd/tbhbdev` | `net.$F(...)`, `http.$F(...)`, and any import of `factory/cli/internal/selfupdate/io/**` |
| `one-http-client` | everywhere except `factory/cli/internal/ghapi/io/**` | `http.Client{...}` and `&http.Client{...}` |
| `no-alpha` | all Go sources except `factory/cli/internal/dispatch` | the string literal `"alpha"`, so no Go code names an `alpha` command (D5) |

Three more rules, `no-cue-binary`, `cue-one-context`, and `no-cobra-globals`, stay in the monorepo's own `sgconfig.yml`, since they hold this module's stack ([appendix K](K-go-stack.md#the-rules-that-hold-it)).

### Test tiers and time caps

gazelle's `map_kind` directive makes every generated `go_test` one of two kinds (D13):

- **`unit_go_test`**, in pure packages: `size = "small"` and `args = ["-test.timeout=10s"]`, so a unit test target fails past 10 seconds. These tests take plain values and use no fakes.
- **`io_go_test`**, under `io/`: `size = "medium"`, so Bazel stops it at 300 seconds. These tests run real tools in a temporary root with a scrubbed environment from one helper, and match fakes by request content.

A `genquery` test fails on any `go_test` in the module that is not one of those kinds, or that sets its own `timeout`. With the targets in parallel, the Go suite's wall time stays under 5 minutes, and tbhb-agent tracks it across repositories as separate CI speed work.

### The front budget

- `go_arch_lint_test` keeps the front's main to its wiring package, and the wiring to the four packages in the components table, plus the standard library. depguard's `front-stdlib-only` list holds the same line for third-party packages across all six of the front's packages, and a planted fixture that imports cobra into the wiring package fails it.
- A size test fails if the stripped front binary grows past a budget set from the first build (new, value to be measured).
- A test runs the front under a process counter and fails if it starts any process before its exec. The front's only reads are the manifest and the self-update cache, plus the shared lock on its bundle.
