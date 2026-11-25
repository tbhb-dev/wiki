---
type: Design Appendix
title: tbhbdev Go port alternatives
description: The options the tbhbdev Go port considered and didn't take, and why.
tags: [tbhbdev, go, alternatives, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words busybox kubectl goreleaser cutover cobra viper koanf cuelang ghinstallation githubv -->

# tbhbdev Go port alternatives

Back to [the design](README.md).

## Finish the Python pure-core refactor

Keep tbhbdev in Python and finish moving decisions out of input/output (I/O) modules. The operator stopped that work in favor of the port (D4). The refactor's finished pieces still help, because a pure Python function with plain-data tests ports almost line for line ([appendix H](H-waves.md)).

## One Go binary with every command

A single binary with a full command tree, as gh builds it. It's one artifact and one build, but every call pays every command's start-up cost, and gh lost 12 to 40 ms per call to dependency start-up ([appendix B](B-gh-pitfalls.md#51-dependency-start-up-costs)). It also pulls every command toward one shared dependency object, which became gh's 17-field Factory. Plugin binaries keep the hooks small and let each command port on its own.

## One multi-call binary

One binary with links named `tbhbdev-<name>`, choosing its command from `argv[0]`, as BusyBox does. It saves disk, but every command still links every dependency, and one bad import slows all of them. The bundle's size isn't a constraint at this scale, so the saving buys nothing.

## In-process plugins

Go's `plugin` package, or plugins over a remote procedure call protocol. The first needs matching toolchains and build flags across every binary, and works only on some platforms. The second adds a protocol and a long-lived child for what exec does with none. Neither keeps the inherited descriptors the launcher relies on.

## Plugins found on PATH

Search `PATH` for `tbhbdev-<name>`, as git and kubectl do, and as the outline proposed after the bundle directory. gh shipped this and removed it six days later ([appendix B](B-gh-pitfalls.md#11-path-discovery-dropped-after-six-days)). It also lets any file on `PATH` run under hooks that agents trigger on every tool call. D3 asks whether listed absolute paths may join the bundle's plugins, which keeps the gain without the search.

## A child process instead of exec

The front starts the plugin, waits, and copies its exit code, as gh runs extensions. That keeps the front alive for cleanup, but the plugin gets a new process ID, signals need forwarding, and every exit path needs wrapping. gh lost exit codes on one path and leaked its own subprocesses' codes on another ([appendix B](B-gh-pitfalls.md#73-exit-codes-from-the-wrong-process)). Exec avoids all three (D9).

## Longest-match nested plugin names

Resolve `tbhbdev alpha agent exec` to `tbhbdev-alpha-agent`, then `tbhbdev-alpha`, as kubectl does and as the outline proposed. One binary per top-level word is simpler to resolve and to test, and with no `alpha` in Go, the nesting the outline needed is gone.

## Exact version matching with a lockstep cutover

Keep `same_version`, stamp one version into every binary, have the front refuse a plugin whose stamp differs, and move the whole fleet at once, as the outline proposed. A half-upgraded fleet would then refuse every run, which D6 and D14 rule out. The release test that checks every stamp catches a mismatch before a bundle ships, so the front has no reason to refuse at run time.

## Freezing the wire protocol for the port

Keep envelope `v` 1 and request `v` 2 fixed until every host runs Go, as an earlier draft of this design decided in D12. With 47 commits to `run_host.py` and `decisions/argv.py` in one month, a freeze would either hold back real changes or tempt unversioned ones. D16 gives those changes a versioned path instead.

## Versions in code instead of a schema file

Keep the version constants in `model.py` and in the Go port, with the corpus as the only contract. Nothing would then tie a change in the request's shape to a version decision, and the two languages could drift. One schema file gives the CI check one thing to compare.

## Readers that refuse unknown fields

Keep today's rule that a reader refuses any field it doesn't know. Then every added field is breaking, and every change needs a bump. Lenient readers with strict writers let an additive change reach older hosts, and the strict writer still catches an undeclared field.

## No self-update, with sync only

Rely on event-driven sync to install every release. A machine that's asleep, offline, or new when sync fires stays behind until the next merge, and a gap of two breaking versions becomes possible. Self-update closes that gap at the machine's next start (D15).

## Overwrite the binary in place

Download the new front and write it over the old file, as many single-binary updaters do. A concurrent run could then exec a half-written file, and a run with Python modules from the old bundle could load new ones mid-run. Renaming whole directories and links never changes a file a run uses.

## Self-update in a background goroutine

Check for updates while the command runs and swap afterward. gh learned that exit then waits on the check ([appendix B](B-gh-pitfalls.md#53-exit-waiting-on-background-work)), and after exec the front isn't there to finish it. The check runs first, in the foreground, under fixed deadlines.

## The fallthrough found on PATH

Have the front run a `tbhbdev-py` it finds on `PATH`, as the outline proposed. A `tbhbdev-py` found that way could come from a different tools commit than the bundle pins, or from an old uv install beside it. The operator chose the same entry point in the front's own folder instead, run by absolute path (D25).

## The bundled interpreter as `python3 -m tbhbdev`

Have the front run `py/bin/python3 -m tbhbdev` by absolute path, as an earlier draft of this design did. It avoids a script's first line, but the operator chose the `tbhbdev-py` entry point in the front's own folder (D25). That keeps every route the front takes a file in `bin/`, and the relocatable environment keeps the script's first line valid after the rename.

## A Go bridge to Python `alpha` commands

Have Go code run `tbhbdev alpha` commands as subprocesses until they port, as the outline proposed for the launcher. That puts `alpha` names in Go code, which D5 rules out, and ties Go behavior to Python argv parsing. The wave 1 review code instead starts runs through the final name `tbhbdev agent`, which Python answers until wave 2, and the launcher's wave ports the library functions it needs, with no command yet.

## The launcher first

Port the launcher in wave 1, as the first draft did, because its `decisions/` package is already pure. The launcher works today, while the review tooling is the part still unbuilt, so the operator put review first (D17). Building the unbuilt review pieces in Go also keeps them out of Python code that the port would replace.

## Cobra in the front

Build the front's command tree with cobra, as gh does. The front must pass unknown names and flags through to a plugin or to Python before anything parses them, and a flag parser consumes or rejects flags it doesn't define. gh turned flag parsing off and then matched error text to work around it ([appendix B](B-gh-pitfalls.md#14-matching-the-unknown-command-error-string)), so the front stays on the standard library (D18).

## Viper for config

Load config with viper, which many cobra programs pair it with. Its usual entry points act on a package-level default instance (unverified), which the no-globals rule bans. koanf loads into a local that the shell owns (D18).

## Running the pinned `cue` binary from Go

Keep Python's approach: install a pinned `cue` and run `cue vet` as a child process. Go has `cuelang.org/go`, which validates in-process with no download, no child, and no I/O in the check itself, so the schema check stays pure (D20).

## Go aliases for alpha names

Let the Go plugins answer to `alpha` names as aliases. That keeps two names alive in Go with no end date, and D5 rules it out. The Python forwarder carries the old names, logs their use, and goes away once the logs stay empty.

## The Go code in tools

Build the Go module in tools beside the Python, as an earlier draft of this design decided in D8, so one tools commit holds both. The operator chose the monorepo tbhb-dev/tbhbdev instead (D22). One monorepo commit still builds one bundle, because it pins the tools commit of its wheel, and byte checks against that commit keep the shared schemas from drifting.

## A Go workspace with many modules

`go.work` with a module per plugin. afakt and awrel use one module each, gazelle handles one prefix well, and plugins share the contract library. One module keeps one `go.sum` and one dependency review.

## A goreleaser or plain Go build path

gh builds with a Go task runner and releases with goreleaser. tbhb-dev builds Go with Bazel in afakt and awrel, with stamps, linters, coverage, and mutation tests already in tbhb_rules_go. A second build path would duplicate all of it.

## A pure-package naming scheme instead of directories

Mark pure packages by a suffix or by a comment that a linter reads. A directory rule, `io/` for I/O, is one glob in every checker, and anyone can see a package's class in its import path.

## githubv4 for GraphQL

Use `githubv4`, the typed GraphQL client, for the calls REST lacks. Those are a few: marking a pull request ready, the merge queue state and enqueue, and Projects v2 by node ID. A small helper on the go-github client covers them with one transport and one token path, and adds no second client to keep current (D24).

## ghinstallation for app tokens

Use `ghinstallation` to mint app installation tokens. It pins go-github v88, which would hold the module five majors behind or put two go-github majors in one build. A JSON Web Token from the standard library and go-github's own installation calls cover the same ground and keep today's lookup, cache, and cooldown (D24).

## Following go-github's latest major

Bump go-github each time it releases a major. Six of the eight majors from v86 to v93 broke calls this code uses. The compiler finds each break, but each one is a pull request at a time tbhb-dev didn't choose, so majors move on tbhb-dev's schedule, and the client uses only constructor options present in both the pinned major and go-github's main branch (D24).
