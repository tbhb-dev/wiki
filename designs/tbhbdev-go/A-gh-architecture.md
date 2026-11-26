---
type: Design Appendix
title: tbhbdev Go port gh architecture
description: How the GitHub command-line tool gh is built, read from its source, and what the tbhbdev Go port copies or avoids.
tags: [tbhbdev, go, gh, research, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words ghcmd cmdutil iostreams ldflags goreleaser goarch goos httpmock moq testscript txtar shlex runF iprompter mousetrap ssoHeader colorable safeexec findsh ghdebug updateable nfpms Codespaces gochecksumtype gomoddirectives nolintlint nosprintfhostport asasalint bidichk bodyclose copyloopvar durationcheck exptostd fatcontext gocheckcompilerdirectives gocritic goprintffuncname ineffassign nilerr gomodguard errcheck staticcheck gosec depguard SSOURL termio golangci -->

# tbhbdev Go port gh architecture

Back to [the design](README.md).

This appendix summarizes a read of the cli/cli source at commit `b546819fd547ef8c97da470bb00f2aaad4165c84` (2026-11-24, module `github.com/cli/cli/v2`). A research run read the code, not the docs, and counted with grep over non-test files, so counts are approximate. Every path and number below comes from that run's report. This design did not read cli/cli again, so treat each as the research run's finding.

## Entry and exit codes

`cmd/gh/main.go` only calls `ghcmd.Main()` and exits with its code. All startup lives in `internal/ghcmd/cmd.go` (508 lines), which keeps it testable.

The exit codes sit in `internal/ghcmd/cmd.go:41-49`: 0 for success, 1 for an error, 2 for a cancel, 4 for an auth failure, and 8 for pending. One if-else chain (`cmd.go:193-243`) maps the typed errors from `pkg/cmdutil/errors.go` to those codes. An extension's own exit code passes through as `*root.ExternalCommandExitError`.

Before any command runs, `Main` loads config, builds the terminal streams, resolves its own path, starts telemetry, builds the Factory, starts an update check, and builds the whole command tree, extensions included. A config error is only a warning at that point, yet `gh version` still never runs on one (see the Factory below).

**For tbhbdev:** copy the one-line `main` and the single error-to-exit map. Keep eager startup work out of dispatch.

## The shared dependency object

`pkg/cmdutil.Factory` has 17 fields (`pkg/cmdutil/factory.go:17-52`): three plain values, six eager objects, and eight function-valued "lazy" fields. `pkg/cmd/factory/default.go:29` builds it.

- The laziness is mostly nominal. `extensionManager(f)` (`default.go:343`) calls `f.Config()` and `f.HttpClient()` during construction, so both run on every call, and HTTP clients are not memoized.
- It is mutable shared state. `root.go:165` copies the whole Factory to change `BaseRepo`, and `cmdutil.EnableRepoOverride` (`pkg/cmdutil/repo_override.go:65-72`) rewrites `f.BaseRepo` in a pre-run hook.
- Every one of 216 `NewCmd*` constructors receives the whole pointer, while 9 fields are each used by 6 or fewer leaf directories.
- Package globals sit beside it: `factory.SSOURL()` reads `ssoHeader` (`default.go:26`), and `root.HasFailed()` reads a flag the help function sets (`pkg/cmd/root/help.go:54`).
- A counter-example inside gh: `pkg/cmd/codespace/common.go:40` takes narrow dependencies explicitly.

**For tbhbdev:** no Factory. Each command group's `io/app` package builds only what that group uses, gives each cobra command a private struct of its own dependencies, and passes narrow values to pure code ([appendix K](K-go-stack.md#explicit-dependencies-and-no-shared-object)).

## Terminal streams

`pkg/iostreams` holds the input, output, and error streams, terminal detection overrides, color, pager, and spinner state. It imports nothing else from the module, which makes it a clean leaf. `iostreams.Test()` returns buffer-backed streams with every terminal flag off, and 227 of 410 test files use it.

**For tbhbdev:** copy the leaf shape and the `Test()` constructor as `internal/termio`. Leave out the pager and spinner until a command needs them.

## Extensions

- **Discovery** (`pkg/cmd/extension/manager.go`): only entries prefixed `gh-` in `config.DataDir()/extensions`. No `PATH` search. Three on-disk shapes exist: a directory with `manifest.yml` for binaries, a git clone, and a symbolic link or path-in-file for local installs.
- **Dispatch** (`Manager.Dispatch`, `manager.go:93-138`): argv goes through verbatim, since the command sets `DisableFlagParsing: true` (`pkg/cmd/root/extension.go:77`). The child gets `GH_EXTENSION=1` and `GH_PATH`, inherits the streams, and runs under `cmd.Run()`, with no signal forwarding.
- **Exit codes:** `pkg/cmd/root/extension.go:43-51` wraps the child's `*exec.ExitError`. `gh extension exec` (`pkg/cmd/extension/command.go:559-564`) does not wrap it, so that path exits 1 and loses the child's code.
- **Names** (`pkg/cmd/root/root.go:195-252`): built-ins register first, and an extension whose name a built-in already takes is skipped without a word. An install-time check (`checkValidExtension`, `command.go:694`) refuses names that clash at install time.
- **Completion:** extension names complete, and their arguments get nothing.
- **Startup cost:** every call lists the extension directory and builds a command for each extension.

**For tbhbdev:** avoid gh's extension architecture (D3). The Python fallback gets verbatim argv and the path back to the front.

## How commands are tested

The clean pattern is `pkg/cmd/label/delete.go`: a private options struct with narrow dependencies (a one-method local interface for the prompter), a constructor that takes a `runF` seam, and a `deleteRun` that does the work. Its tests come in two tables, one for flags and one for the run.

| Helper | Test files, of 410 |
| --- | --- |
| `iostreams.Test()` | 227 |
| `cmdutil.Factory{` literals | 180 |
| `httpmock.Registry` | 175 |
| `runF` capture | 131 |
| `t.TempDir` | 80 |
| `t.Setenv` | 64 |
| `run.Stub()` for exec | 31 |

`internal/run/run.go:24` keeps a package variable, `PrepareCmd`, that `run.Stub()` swaps. With `t.Setenv` as well, only 16 test files run in parallel. `pkg/cmd/pr/create/create_test.go` runs 3,580 lines for a 1,409-line command, because its decisions are tangled with git and HTTP calls.

The extension manager is the good counter-example. `manager.go:45` injects `lookPath`, `newCommand`, `dataDir`, and `platform`, and `manager_test.go:27,48` re-runs the test binary as a fake child.

**For tbhbdev:** decisions are pure and tested with plain tables. Adapters inject their process and file functions, as the extension manager does, and use the helper-process pattern for real children.

## Package structure

- `pkg/cmd/factory` imports `pkg/cmd/extension`, so the plugin manager lives inside a command package.
- The "public" `pkg/` tree imports `internal/`, so nobody outside the module can reuse it.
- Commands import each other's `shared` packages, such as `issue` to `pr/shared` and `run` with `workflow` both ways.
- `.golangci.yml` has no depguard, gomodguard, or architecture rule. Only Go's `internal/` visibility and its cycle ban guard the layers.

**For tbhbdev:** `go_arch_lint_test` and depguard run from the first commit ([appendix F](F-build-and-checks.md)). Domain packages reach another domain only through a package the arch config names as shared.

## Build, release, and version

`script/build.go:53-65` builds with `-ldflags "-X .../internal/build.Version=<v> -X .../internal/build.Date=<date>"`, taking the version from `GH_VERSION`, then `git describe`, then the short commit. `internal/build/build.go` falls back to `debug.ReadBuildInfo()` for `go install` builds. `.goreleaser.yml` stamps the same symbols for every platform.

The update checker (`internal/update/update.go`) runs only in builds with the `updateable` tag, which the Homebrew formula sets. It checks at most once a day and never when output is not a terminal.

**For tbhbdev:** one stamp package, as awrel's `buildmeta` already does with Bazel `x_defs`, and a release test for the bundle version. The update check at start is one cache read and a pure decision, with any network work in the internal self-update helper ([appendix J](J-self-update.md)).

## What this design copies and what it avoids

| gh practice | Where in gh | tbhbdev |
| --- | --- | --- |
| One-line `main`, startup in a package | `cmd/gh/main.go`, `internal/ghcmd/cmd.go:51` | copy |
| Typed errors mapped to codes in one place | `pkg/cmdutil/errors.go`, `cmd.go:193-243` | copy, and exec preserves the Python fallback status |
| Dispatched marker and path back | `manager.go:134`, `cmd.go:428` | pass `TBHBDEV_PATH` to the Python fallback |
| Verbatim argv | `pkg/cmd/root/extension.go:77` | copy |
| Built-ins win, with an install-time check | `root.go:197-207`, `command.go:694` | keep Go command names ahead of the Python fallback, with the checks in [appendix G](G-names-and-migration.md#collisions-with-python-names) |
| Leaf streams package with `Test()` | `pkg/iostreams/iostreams.go` | copy |
| Options struct, `runF` seam, one-method interfaces | `pkg/cmd/label/delete.go` | copy in `io/app` packages |
| Injected process and file functions | `pkg/cmd/extension/manager.go:45` | copy in adapters |
| One stamp package with a build-info fallback | `internal/build/build.go` | copy |
| A god Factory | `pkg/cmdutil/factory.go` | avoid |
| Mutating shared dependencies in hooks | `repo_override.go:70`, `root.go:165` | avoid |
| Package-level mutable state | `help.go:54`, `default.go:26`, `run.go:24` | avoid |
| Eager work before dispatch | `cmd.go:57-174`, `default.go:343` | avoid |
| A plugin manager inside a command package | `pkg/cmd/factory` to `pkg/cmd/extension` | avoid |
| Implicit discovery shapes | `manager.go:~150-190` | avoid extension discovery |
| Layering by convention only | `.golangci.yml` | avoid |
| Regex stubs of git command lines | `pkg/cmd/pr/create/create_test.go` | avoid |
