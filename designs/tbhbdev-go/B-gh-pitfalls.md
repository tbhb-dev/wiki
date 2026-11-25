---
type: Design Appendix
title: tbhbdev Go port gh pitfalls
description: The architectural pain in the history of the GitHub command-line tool gh, by commit, each with the rule it gives the tbhbdev Go port and where the design applies it.
tags: [tbhbdev, go, gh, research, history, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words ghcs ldflag ldflags Lstat cygwin tcell runewidth tzutil keyring safeexec mintty moq Homebrew Codespaces GHES ghdebug updateable textconv golangci gochecknoglobals termio gochecknoinits koanf cmdutil -->

# tbhbdev Go port gh pitfalls

Back to [the design](README.md).

A research run read the full history of cli/cli on `trunk`, with HEAD at `b546819fd547` (2026-11-24) and 12,685 commits, using `git log` message searches, pickaxe searches, blame, and the diffs. Every commit hash, date, and path below comes from that report, and the run marked as *inferred* anything a commit message or diff does not state. Issue numbers appear only where a commit message carries them.

Each entry ends with the rule it gives this design and the place the design applies it.

## 1. Discovery and names

### 1.1 PATH discovery dropped after six days

`fce93d60809b` (2021-05-28) made every executable named `gh-*` on `PATH` into a gh command. `4bdddd72d34b` (2021-06-03) removed the `PATH` search and made local extensions explicit symbolic links. The reason is not stated (*inferred*: unbounded, ambient discovery).

**Rule:** plugins come from the bundle's own directory, never from `PATH`. **Applied in:** [E, dispatch](E-plugin-contract.md#dispatch), and D3 for anything outside the bundle.

### 1.2 Name clashes caught only at install

`103e18cab548` and `0482e5cd9b6c` (2021-07-01) refused installs whose name matched a built-in or an installed extension. Names come from stripping `gh-`, so `a/gh-foo` and `b/gh-foo` clash.

**Rule:** a plugin's name is its top-level command word, unique in the manifest, and a bundle build fails on a duplicate. **Applied in:** [E, the manifest](E-plugin-contract.md#the-manifest).

### 1.3 A new built-in silently shadowed an extension

`10b4a1f4272c` (2026-01-19) made root registration skip any extension whose name a built-in already resolves, after the built-in `copilot` command arrived the same day (`39880650d5c0`, a link that is *inferred*). The user gets no warning.

**Rule:** routing is one pure function over built-ins, bundle plugins, outside plugins, and the fallthrough, with a fixed order, and a collision is reported, never skipped. A Go plugin may shadow a Python top-level name only when it covers every subcommand of it. **Applied in:** [E, dispatch](E-plugin-contract.md#dispatch), and [G](G-names-and-migration.md#collisions-with-python-names).

### 1.4 Matching the unknown-command error string

`75909ed2ba1b` (2026-04-15) suggested official extensions by matching cobra's error text, which caught flag values and arguments after `--` and gave wrong exit codes. `7ad1d7c0a16b` (2026-04-16) replaced it with hidden stub commands.

**Rule:** route on parsed structure, never on another program's error text. An unknown name falls through to Python, and Python's own usage error comes back unchanged. **Applied in:** [E, dispatch](E-plugin-contract.md#dispatch).

### 1.5 Five fixes to find its own path

`98f1f5ec0d57` (2021-03-03) wrote an absolute path into git config because Homebrew's git ran without `/usr/local/bin`. `2dba58bfa80b` (2021-08-30) and `d731cb9c73d7` (2021-09-06) walked `PATH` because `os.Executable()` returned a Cellar path that upgrades delete. `7387346196a6` (2022-06-20) swapped `os.Stat` for `os.Lstat`, `05bf29d88ec4` (2023-03-07, #7025) added a `GH_PATH` override for cygwin, and `5e14987d65a7` (2023-11-14, #8238) fixed the same function again. `d853ce5bc980` (2021-09-24) and `3cce16e72d76` (2021-12-21) made the lookup lazy.

**Rule:** only the front resolves its path, once, with `os.Executable` and `EvalSymlinks`, and passes it and the bundle root to every plugin. The bundle lives in a versioned directory that an upgrade never deletes while a link points at it or a run holds it open. The path logic is a pure function with table tests. **Applied in:** [E, the contract](E-plugin-contract.md#the-plugin-contract), and the installer in [E](E-plugin-contract.md#install-and-the-bundle-directory).

### 1.6 Five years without a dispatch contract

`408534a1be91` (2026-08-05) set `GH_EXTENSION=1` for extensions, and `30419651a944` (2026-08-28) added `GH_PATH`. Until then an extension could not tell gh started it, or call the same gh back.

**Rule:** the contract's environment is in version 1, carries its own version number, and has a test per variable. **Applied in:** [E, the contract](E-plugin-contract.md#the-plugin-contract).

### 1.7 State keyed by a name that is not unique

`97630fe73c1c` (2024-12-08) and `0a1c9be65699` (2025-01-10) cleared per-extension update state on install and removal, because state keyed by name could pass from a removed `a/gh-foo` to a new `b/gh-foo`.

**Rule:** the bundle version keys all install state, and a plugin keeps no install state of its own. **Applied in:** [E, install](E-plugin-contract.md#install-and-the-bundle-directory).

### 1.8 Three spellings of one name

`1881de6d4191` (2021-08-19) and `00e97121ec8c` (2022-05-17, #5464) normalized `foo`, `gh-foo`, and `owner/gh-foo`.

**Rule:** one spelling per command, the final name, and the old `alpha` spelling lives only in the Python forwarder. **Applied in:** [G](G-names-and-migration.md).

## 2. Versions and upgrades

### 2.1 A renamed symbol left releases saying DEV

`810c127608a2` (2021-09-10) fixed `ghcs`, a second binary from the same repository, whose renamed `Version` variable made the `-X` flag do nothing. Released builds printed `DEV`. Nothing guards a repeat (*inferred*).

**Rule:** every binary links the same stamp package, and a release test runs each one with `--version` and fails unless it prints the bundle version. **Applied in:** [F, checks](F-build-and-checks.md#checks), and [E, version stamp](E-plugin-contract.md#the-version-stamp).

### 2.2 Two version sources

`de6e99aa75c0` and `408b565fd960` (2020-02-18) fell back to `debug.ReadBuildInfo()` when no flags were set, keeping flags as the override. Builds made that way have no date.

**Rule:** Bazel stamps every release binary, and the build-info fallback only marks a developer build as such. **Applied in:** [E, version stamp](E-plugin-contract.md#the-version-stamp).

### 2.3 Comparing describe strings as versions

`82d19b73d63d` (2021-01-20) stopped telling source builds to upgrade to a release they were already past, since `v1.4.0-34-g<sha>` sorted below `v1.4.0`.

**Rule:** build comparison is a pure function with table tests, and only logging uses it, never a refusal. Self-update compares release numbers from the stamp, never describe strings or commit times. **Applied in:** [I, the transition](I-protocol.md#the-transition), and [J, the decision](J-self-update.md#the-decision).

### 2.4 An update checker that fought package managers

`feb4acc2c00c` (2021-02-08), `27aea42d8a43` (2021-02-23), `2097899a4dff` (2023-02-06), and `22504bfa96de` (2025-05-26) moved from Homebrew hints to a 24-hour delay to a compile-time opt-in.

**Rule:** no package manager installs tbhbdev, so its own update path fights nothing. The check at start is one cache read and a pure decision, the network work runs in a separate helper, and `TBHBDEV_NO_SELF_UPDATE=1` turns it off. The fleet check, `tbhbdev alpha version-check` until wave 4 ports it as `tbhbdev install check`, still compares hosts on request. **Applied in:** [J, self-update](J-self-update.md), and [G](G-names-and-migration.md).

### 2.5 Waiting on a release check after the command

`47d77bd51b58` (2024-11-02) checked extension releases after each run, `b8ef951de1cf` (2024-11-13) cut the wait from 3 s to 1 s, and `bc8c46b0b1c1` (2025-01-13) made it non-blocking.

**Rule:** the front does no network work, and after exec it is not there to do any. Self-update's network work runs before the command, in its own helper, under fixed deadlines. **Applied in:** D9, [E, dispatch](E-plugin-contract.md#dispatch), and [J](J-self-update.md#failures).

### 2.6 Shared code for checks that diverged

`0d3f7cae4e2a` (2024-11-17) joined the core and extension update checks, and `20062233c840` (2025-01-10) split them again.

**Rule:** share a package only when its callers need the same behavior, and keep plugin domains apart. **Applied in:** [F, layering](F-build-and-checks.md#checks).

### 2.7 A refactor broke pinning for 21 months

`d7277e396cfc` (2022-03-01) and `33e589465e0d` (2022-11-03, #6513) stored pins differently per extension kind. The lazy-loading refactor `beb6234f96a6` (2023-06-14) built the git pin path from the executable instead of its directory, and `e02ee18ed2b2` (2025-03-15) fixed it.

**Rule:** one plugin kind, one on-disk layout, one manifest format. **Applied in:** [E, the manifest](E-plugin-contract.md#the-manifest).

### 2.8 Extensions that change kind

`54b82dd0726b` (2021-10-21) reinstalled git extensions that started to ship binaries, because the kind came from which files existed.

**Rule:** the manifest states each entry's kind, and the installer never infers it. **Applied in:** [E, the manifest](E-plugin-contract.md#the-manifest).

## 3. The shared dependency object

### 3.1 From 2 fields to 17

`bef62faaead7` (2020-05-20) created the Factory with two fields. It grew to 6 by `8ff4cc40e642`, 9 by `354bac82b3a5`, 11 by `6618baaf63cb`, 12 by `541ce0e5b492`, 13 by `5761a1e66965`, and 17 by `663eda0c3285` (2026-11-24), the 17 fields that [appendix A](A-gh-architecture.md#the-shared-dependency-object) counts in `pkg/cmdutil/factory.go`. Nobody has split it.

**Rule:** no shared dependency object. Each plugin's `io/app` builds what that plugin uses, and each cobra command's constructor takes only its own dependencies. **Applied in:** [E, library surface](E-plugin-contract.md#the-library-surface), and [K, explicit dependencies](K-go-stack.md#explicit-dependencies-and-no-shared-object).

### 3.2 Flags read before parsing

`c82e38353fd0` (2019-10-18), `8370602f4910` (2019-10-25), and `5e24e0d9b9d7` (2020-08-07) removed package-level state that read flags at `init()` time, which tests swapped by hand.

**Rule:** no package-level mutable state and no `init()`. golangci-lint's `gochecknoglobals` and `gochecknoinits` run on every package, and only the stamp package carries a globals exception, as awrel's `buildmeta` does. koanf loads config once in `io/app`, after flags are parsed. **Applied in:** [F, checks](F-build-and-checks.md#checks), and [K, config](K-go-stack.md#config-with-koanf).

### 3.3 Commands reading the environment

`111e8dbcf29d` (2021-03-19) passed a browser to each command, since commands read `BROWSER` themselves and tests depended on it.

**Rule:** pure code never reads the environment. `io/app` parses it once into a struct. **Applied in:** [F, ast-grep rules](F-build-and-checks.md#ast-grep-rules).

### 3.4 Four HTTP client fields

`2794f7b8d631` (2025-10-13), `e6dfcd3ce728` (2026-05-27), and `24968b6ab2f7` (2026-11-18) added client fields that differ only by middleware.

**Rule:** one constructor per kind of client in one adapter package, with options. **Applied in:** [E, library surface](E-plugin-contract.md#the-library-surface).

### 3.5 A command that built its own client

`268453803ee0` (2026-03-26) and `4a75987727d9` (2026-09-03) fixed `gh api`, whose inline client lost the agent User-Agent suffix and the telemetry opt-out.

**Rule:** an ast-grep rule forbids `http.Client{` outside the GitHub adapter. **Applied in:** [F, ast-grep rules](F-build-and-checks.md#ast-grep-rules).

### 3.6 Lazy in name only

`17776cafc17d` (2026-04-17) left a comment that `Factory.Config` is lazy, yet root setup calls it and fails, so `gh version` never runs on a config error.

**Rule:** `tbhbdev version` and `tbhbdev help` read only the manifest. A plugin reads config only inside a subcommand that needs it. **Applied in:** [E, dispatch](E-plugin-contract.md#dispatch).

### 3.7 Two ways to get one value

`2a99ed9afb45` (2020-07-23) kept both a remotes-only and an API-resolved base repository as function fields.

**Rule:** resolve shared context once, in `io/app`, and hand pure code the value. **Applied in:** [E, library surface](E-plugin-contract.md#the-library-surface).

## 4. Tests, global state, and I/O

### 4.1 Order-matched fakes

`1d4065e4c48f` (2020-05-06) and `0a4d4ee0074c` (2020-05-07) replaced an HTTP fake that answered in order, and `f9783fe812da` (2020-09-23) did the same for exec stubs.

**Rule:** adapters match fakes by request content. Pure code needs no fakes. **Applied in:** [F, test tiers](F-build-and-checks.md#test-tiers-and-time-caps).

### 4.2 Tests that read the developer's machine

`183db9953578` (2019-10-17) stopped tests reading the developer's Secure Shell (SSH) config file. `c430da732960` (2023-10-19), `08c7bd1df2ef` (2023-11-30), and `c9a6b7cc00d0` (2023-12-01) isolated config and the keyring. `e5b9c168fc83` (2026-08-10) cleared `GH_TOKEN` and friends after nine packages failed, and `5e8ec5d57fd1` (2026-08-10) scrubbed `GIT_CONFIG_*` too.

**Rule:** pure tests never see the real environment, which ast-grep enforces. Adapter tests get a scrubbed environment and a temporary root from one helper. **Applied in:** [F, ast-grep rules](F-build-and-checks.md#ast-grep-rules).

### 4.3 Synthetic git output

`bc612ed3d618` (2026-10-29) and `934db7e6623e` moved git tests to real temporary repositories and deleted the seams.

**Rule:** adapter tests run the real tool in a temporary root, under the 5-minute cap. **Applied in:** [F, test tiers](F-build-and-checks.md#test-tiers-and-time-caps).

### 4.4 The clock and test hooks in releases

`a9bdd8d186b6` (2023-12-13) fixed tests tied to relative dates, and `d00a58660e78` (2026-10-19) moved test-only environment hooks behind a debug build tag.

**Rule:** pure code takes the time as an argument, and ast-grep bans `time.Now` there. No test hook ships in a release binary. **Applied in:** [F, ast-grep rules](F-build-and-checks.md#ast-grep-rules).

## 5. Startup time

### 5.1 Dependency start-up costs

`8741b648a64f` (2023-05-26, #7487) cut 30 to 40 ms from a dependency's `init()`, and `2d3d20f04058` (2026-10-02) cut about 12 ms with a dependency bump.

**Rule:** the front imports only routing, contract, and stamp packages, which arch-lint enforces, under a size budget. **Applied in:** [F, the front budget](F-build-and-checks.md#the-front-budget).

### 5.2 Per-extension work on every call

`beb6234f96a6` (2023-06-14, #7493) made extension listing lazy, and `731e9b50e46a` (2026-10-07, #14620) stopped spawning git for each extension at start.

**Rule:** the front reads one manifest and one self-update cache file and spawns nothing before its exec, which a test checks. **Applied in:** [F, the front budget](F-build-and-checks.md#the-front-budget).

### 5.3 Exit waiting on background work

`626c639df5e9` (2023-02-07, #6978) stopped exit from waiting on the update goroutine. `79f749a02b18` (2020-04-07) and `c63247f0eac8` (2021-03-04) kept checks out of completion and the git credential helper.

**Rule:** the dispatch path starts no background work, which the hooks need most, since they run on every tool call. Self-update runs in the foreground before the command, never in a goroutine that exit waits on. **Applied in:** [E, dispatch](E-plugin-contract.md#dispatch).

### 5.4 Lookups in a slow order

`ff4e9aad5629` (2026-10-16) probed the usual keyring slot first, saving about 30 ms per request on macOS.

**Rule:** a credential adapter caches within one run and probes the common case first. **Applied in:** [H, waves 1 and 2](H-waves.md#waves).

### 5.5 A child process on every call

`3d9f22c115ac` (2026-05-09) fixed a console flash from the telemetry child that every call starts on Windows.

**Rule:** telemetry writes to the state directory in-process, and no command starts a detached child. **Applied in:** [E, library surface](E-plugin-contract.md#the-library-surface).

## 6. Auth and config

### 6.1 One interface for config and tokens

`a325db305190` (2020-04-14), `cacff4ad6d57` (2022-06-23, #5771), `a33e12a21de7` (2023-02-28, #7033), `792487831557` (2023-10-27, #8246), and `1d382306755e` (2024-05-10, #9060) reshaped the config interface five times.

**Rule:** settings and credentials are separate adapters. **Applied in:** [E, library surface](E-plugin-contract.md#the-library-surface).

### 6.2 Each command resolved the host

`9ec1e21d4cf2` (2021-03-30, #3286), `69ca2dda4a49` (2021-04-02), `7ec2b2251d66` (2022-04-08), `22a5a4c899ce` (2024-08-02), and `c08895194408` (2024-08-02) fixed host handling one command at a time.

**Rule:** resolve repository and host context once per run in `io/app`. **Applied in:** [E, library surface](E-plugin-contract.md#the-library-surface).

### 6.3 Writes to a read-only config directory

`8dbd07212c23` (2022-04-11, #5378) stopped login rewriting all of `config.yml`, and `cbac639fc160` (2026-10-16) moved a lock into the state directory, since the config directory can be read-only.

**Rule:** tbhbdev never writes the config directory. Logs, locks, and forwarder records go to `$XDG_STATE_HOME/tbhbdev`. **Applied in:** [G, the forwarder](G-names-and-migration.md#the-forwarder), and D3.

### 6.4 A timeout too short for a person

`0ca080da51a7` (2026-07-03) raised the keyring timeout from 3 s to 60 s.

**Rule:** timeouts are named values passed into pure code, with a test per value. **Applied in:** [H, wave 2](H-waves.md#waves).

### 6.5 A migration on the hot path

`d2ff55737cf4` (2023-11-01) ran a config migration on every call, where an error stopped any command, until `ba51bb47799e` (2026-09-11, #14430) removed it.

**Rule:** migrations run only from an explicit command, never during dispatch. **Applied in:** [E, dispatch](E-plugin-contract.md#dispatch).

### 6.6 A public seam made of internal types

`7fea69a1fd08` (2026-10-12) restored a config interface that external modules could no longer implement.

**Rule:** the plugin contract is environment variables and a manifest, not Go types, so outside plugins never import monorepo packages. **Applied in:** [E, the contract](E-plugin-contract.md#the-plugin-contract).

### 6.7 A command resolving credentials itself

`604120a8cf26` (2026-10-13) fixed a token lookup keyed on the resolved user instead of the flag.

**Rule:** credential lookup lives in one adapter with table-tested keys. **Applied in:** [H, waves 1 and 2](H-waves.md#waves).

## 7. Exec and processes

### 7.1 LookPath searched the current directory

`5b4a08dcb9bc` (2020-11-10) moved every lookup to `safeexec`, since Go's `exec.LookPath` on Windows searched the current directory.

**Rule:** the front runs plugins by absolute path in the bundle, and ast-grep bans `exec.LookPath` in it. **Applied in:** [F, ast-grep rules](F-build-and-checks.md#ast-grep-rules).

### 7.2 Scripts and Windows

`4bd0435c38b0` (2020-07-14), `acaaa28fd727` (2020-07-15), and `1ec47d8191ca` (2021-06-21) ran scripts through Git for Windows' `sh.exe`. `42ce8faafa85` (2021-10-19) and `6b7889be6e48` (2021-11-12) ran binaries directly and fixed `.exe` names, and `3946606e5e20` (2021-08-11) used path files instead of symbolic links on Windows.

**Rule:** Windows is out of scope (D9), and the front execs the bundle's `tbhbdev-py` entry point by its absolute path in the front's own folder (D25). It never names an interpreter, never runs `python3 -m tbhbdev`, and never searches `PATH`. **Applied in:** [E, the fallthrough](E-plugin-contract.md#the-python-fallthrough).

### 7.3 Exit codes from the wrong process

`cfb8eebf306f` (2020-07-15), `9234163679f2` (2021-03-04), and `9be9dc22e9b0` (2023-06-20, #7567) passed child exit codes through, then had to stop gh's own git exits leaking out as its status.

**Rule:** exec makes the plugin's status the process's status, and the front keeps 125 to 127 for itself. **Applied in:** [E, exit codes](E-plugin-contract.md#exit-codes).

### 7.4 No signal forwarding

The log has no commit about forwarding signals to extensions. Only `22f9824ec8da` (2021-09-09) touches Ctrl-C, for prompts (*inferred*: gh relies on the terminal signaling the whole group).

**Rule:** exec delivers signals straight to the plugin. Where a Go server must run Python as a child, it forwards SIGINT and SIGTERM and exits with the child's status. **Applied in:** [I, the transition](I-protocol.md#the-transition).

## 8. Arguments, help, and completion

### 8.1 Help for plugins

`51d609078b51` (2021-08-19) rewrote `gh help foo` to `foo --help`.

**Rule:** `tbhbdev help <name>` runs `<name> --help`. **Applied in:** [E, dispatch](E-plugin-contract.md#dispatch).

### 8.2 Plugins special-cased before the command tree

`74ee8cacae8f` (2023-05-25, #7457) removed 155 lines of hand-rolled dispatch from `main`. `e3569dc7bde3` (2023-06-02) and `9be9dc22e9b0` followed up on completion and exit codes.

**Rule:** built-ins, plugins, and the fallthrough are rows of one routing table from the first commit. **Applied in:** [E, dispatch](E-plugin-contract.md#dispatch).

### 8.3 Completion stops at the plugin name

`7a3d02df4702` (2022-04-26) added descriptions for extension names. No commit forwards completion requests to an extension.

**Rule:** contract version 1 reserves `__complete` as a first argument for plugins, and the front forwards it. Plugins may implement it later. **Applied in:** [E, the contract](E-plugin-contract.md#the-plugin-contract).

### 8.4 Output streams

`7859afb5866b` (2022-05-25) put completion output back on standard output, and `3f2ae26b409a` (2026-08-19) stopped one usage error splitting across two streams.

**Rule:** results go to standard output, diagnostics and usage errors to standard error, through `internal/termio`. **Applied in:** [E, library surface](E-plugin-contract.md#the-library-surface).

### 8.5 Global flags and plugins

The log has no commit that passes gh's global flags to extensions. `gh foo --repo x` hands `--repo x` to the extension untouched.

**Rule:** the front takes no flags before the command word, so nothing is lost or reinterpreted. **Applied in:** [E, dispatch](E-plugin-contract.md#dispatch).
