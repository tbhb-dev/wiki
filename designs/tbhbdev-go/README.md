---
type: Design Draft
title: Port tbhbdev to Go as plugin binaries
description: How tbhbdev moves from Python to a Go front and per-command plugin binaries in one monorepo, with a Python fallthrough, final command names, and no launch ever refused on a version mismatch.
tags: [tbhbdev, go, cli, plugins, bazel, design]
status: draft
owner: tools maintainer
updated: "2026-11-25"
decisions_open: 3
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words gazelle depguard golangci kubectl syscall buildinfo unported -->

# Port tbhbdev to Go as plugin binaries

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | tools maintainer | 2026-11-25 | 3 open decisions in [decisions.md](decisions.md), about 10 minutes |

## Summary

tbhbdev is about 37,700 lines of Python. On 2026-11-25 the operator chose to port it to Go as plugin binaries in one monorepo, modeled on the GitHub command-line tool `gh`, and to stop the Python pure-core refactor. This design gives the port's shape and its first build.

A small Go front, `tbhbdev`, runs `tbhbdev-<name>` from its own release bundle by exec. It hands every other name, all of `alpha` included, to the Python tbhbdev built from the same commit. Commands move one at a time under final names, and a Python forwarder carries old `alpha` callers until they move.

No version mismatch refuses a launch at any step. Pure packages and input/output (I/O) packages are checked apart from the first commit. Three questions wait on the operator: release signing, what retires, and plugins outside the bundle.

## Proposal

When the first build merges, these parts exist in tbhb-dev/tools, beside the Python under `src/tbhbdev` (D8):

- **The front, `cmd/tbhbdev`** (new). A Go binary that maps argv to one route and replaces itself with that route's program. It does no network, config, or per-plugin work before the exec.
- **Plugin binaries, `cmd/tbhbdev-<name>`** (new). One per top-level command word, such as `tbhbdev-agent` for `tbhbdev agent exec`. Each owns its subcommands and its own small dependency set.
- **The Python fallthrough** (existing code, new place). The Python package from the same commit, installed inside the bundle, runs every name the front doesn't route to Go, including all of `tbhbdev alpha` (D5).
- **The release bundle** (new). One archive per platform, built from one commit, with the front, every plugin, the Python wheel, and a manifest. The tools continuous integration (CI) release job builds it, and the tools maintainer role runs that job.
- **The installer** (new behavior). It unpacks a bundle into its own versioned directory and swaps one symbolic link, so a failed install leaves the old bundle running. The ops maintainer role rolls it onto hosts one at a time.
- **A shared Go library** (new). Small packages for the plugin contract, routing, build stamps, terminal streams, and process runs. [Appendix E](E-plugin-contract.md) lists them.
- **Checks from the first commit** (new, on existing rules). The pull request size gate, an import-direction check that keeps I/O out of pure packages, ast-grep rules for I/O in unit tests, and test time caps. [Appendix F](F-build-and-checks.md) has the layout and each check.

Other roles carry parts of it. The rules maintainer adds the ast-grep test macro to tbhb-dev/rules, the agent-config maintainer moves hook and brief callers to final names, and the CI speed role owns CI wall time across repositories.

## How it works

### Dispatch

1. The front resolves its own path once, from `os.Executable` and `filepath.EvalSymlinks`, and reads the bundle manifest beside it.
2. A pure function, `dispatch.Resolve`, maps argv and the manifest to one route. The order is front built-ins (`version` and `help`), then bundle plugins by the first command word, then the Python fallthrough.
3. The front sets the contract variables and calls `syscall.Exec`. The plugin inherits the process ID, the argv tail verbatim, the environment, and every open descriptor, so the forced command's standard input contract and the inherited push-token pipe survive as they are.
4. The plugin's own exit status is the process's exit status, since nothing waits on it. The front keeps 125, 126, and 127 for its own failures.

The front never searches `PATH` for plugins and never routes `alpha` to Go. Whether it may run plugins from outside the bundle is D3.

### The plugin contract

Each plugin gets the argv after its name, unparsed. It also gets `TBHBDEV_DISPATCHED=1`, `TBHBDEV_PROG`, `TBHBDEV_PATH`, `TBHBDEV_BUNDLE`, `TBHBDEV_BUNDLE_VERSION`, and `TBHBDEV_CONTRACT=1` beside the inherited environment. Every binary carries the same build stamp, linked from one package as awrel's `buildmeta` does, and prints it for a lone `--version`.

The manifest lists each plugin's name, file, and SHA-256 hash, the Python entry, and the contract version. The installer checks the hashes once, and the front trusts the manifest at run time. [Appendix E](E-plugin-contract.md) gives the full contract.

### Versions without refusal

Today a driver refuses a host whose build version differs, through `same_version` in `decisions/driver.py` (existing). The port removes that refusal before any Go ships (D6) and pins compatibility on the wire protocol instead.

| Phase | Hosts run | Launch rule | Refusals |
| --- | --- | --- | --- |
| 1. Last lockstep release | Python, installed as today | A Python release turns the build-version check into a logged warning and checks protocol versions | Older builds still refuse during this one rollout, as during every release today |
| 2. Bundle with Python behind the front | Mixed: phase 1 Python, or a bundle | Both sides run the same Python code for `alpha agent` | None: a build mismatch only logs |
| 3. Go `agent` in the bundle | Mixed, with Go and Python drivers | The wire protocol stays frozen at envelope `v` 1 and request `v` 2 (D12). A Go host hands any request it can't serve to its bundled Python server | None: every host serves every driver |
| 4. Python server removed | Bundles only, forced command on Go | Protocol changes reach hosts first, then drivers | None: no driver sends a version some host lacks |

A shared corpus of request envelopes tests every pairing in CI: a Python driver against a Go host, and a Go driver against a Python host. [Appendix E](E-plugin-contract.md#versions-and-the-no-refusal-transition) walks each phase.

### Pure packages and I/O packages

Pure packages take plain values and return data, as the Python `decisions/` package already does. I/O lives only under `io/` directories, in `cmd/`, and in each plugin's wiring package. No package takes the name `core` or `shell`.

| Check | What it enforces | Status |
| --- | --- | --- |
| Pull request size gate | 400 added code lines and 1,500 raw lines per pull request, Go tests included | existing, rules `pr_size` |
| `go_arch_lint_test` | Pure packages import only pure packages, and `io` imports never point back | existing macro, new config |
| golangci-lint depguard | Pure files import only an allowlist of standard packages | existing aspect, new config |
| `ast_grep_test` | No file, process, network, clock, or environment calls in pure code or its tests | new macro in rules |
| Test caps | A unit test target fails after 10 seconds, and any Go test target after 5 minutes | new wrapper macros |
| Front budget | The front links only routing, contract, and stamp packages, under a size cap | new |

### Bazel layout

The Go module is `go.tbhb.dev/tools` at the repository root, with rules_go, gazelle, and tbhb_rules_go set up as afakt and awrel set them up. Every Go package and the Python suite are separate Bazel packages with disjoint inputs, so a Go-only change leaves `pytest_test` unaffected in pull requests and in the merge queue (D7). [Appendix F](F-build-and-checks.md) has the tree and the input checks.

### Names and callers

Go has no `alpha`. Each command gets its final name when it ports, such as `tbhbdev agent exec` and `tbhbdev pr create`, and [appendix G](G-names-and-migration.md) maps every `alpha` command.

After a command ports, its Python `alpha` entry becomes a silent forwarder to the Go command and logs each call in the state directory. Hooks, briefs, scripts, and agent-config move to the final name, a check stops new `alpha` uses, and the Python code goes once the logs show no calls for 14 days.

### Port waves

Wave 1 ports the agent launcher and the libraries it imports, because its `decisions/` package is already pure. Wave 2 ports the harness hooks, the pull request commands, and `loop`. Wave 3 ports the rest. [Appendix H](H-waves.md) gives the line counts and the parallel slices: about four serial pull requests, then up to eight at once.

### What gh taught

gh got the one-line `main`, the typed exit errors, and the injected process seams right ([appendix A](A-gh-architecture.md)). Its history shows what to avoid: plugins found on `PATH`, silent name collisions, a version stamp that silently didn't apply, five fixes to find its own path, and a 16-field Factory. [Appendix B](B-gh-pitfalls.md) ties each pitfall to a rule here.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| The phase 1 Python release still ships in lockstep | Hosts on older builds refuse each other while it rolls out | It ships the way every release ships today, and it's the last one that must | existing: `same_version` |
| The Go agent server behaves differently from the Python one | A launch fails or runs wrongly on a Go host | The envelope corpus runs both ways in CI, and the Go host hands unknown requests to Python | new |
| The forced command or the push-token descriptor doesn't survive exec | Remote launches break | An integration test per host operating system, in the first build | unverified |
| Timeout and kill behavior drifts from `process_group.py` | Runs hang or die early | Port the Python tests for it first, as Go table tests | unverified |
| Go code runs longer than the Python it replaces | Slices split smaller under the 400-line gate | Size slices by Go lines after the first two | unverified |
| A gap in the depguard or ast-grep rules | I/O slips into a pure test | Each rule has a planted failing fixture | new |
| State files read by both languages drift | One side misreads the other's state | Go reads and writes the Python formats, with shared fixtures | new |
| The remote cache doesn't serve a pull request run | `pytest_test` runs on a Go-only change anyway | The CI speed role picks cache reads or affected-target selection | unverified |
| Library callers in other repositories | Removing Python modules breaks them | They stay in Python until their callers move | unverified |

[Appendix D](D-evidence.md) lists the evidence behind each claim.

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | `ast_grep_test` in tbhb-dev/rules | A rules test fails on a planted `os.ReadFile` in a pure test fixture | rules maintainer |
| 2 | Phase 1 Python release: a warning in place of the build-version refusal, `TBHBDEV_PROG`, and `pytest_test` and the Python linters on Python inputs only | A driver test with mismatched stamps launches, and a commit that adds only a `.go` file reports `//tests:pytest_test` as cached or not run | tools maintainer |
| 3 | Go skeleton: the front, `internal/dispatch`, `internal/plugin`, `internal/buildinfo`, the Bazel setup, and every check in the table | `tbhbdev alpha agent status` through the front prints what the Python command prints, and each check fails on its planted fixture | tools maintainer |
| 4 | The bundle build, the release job, and the installer | On one host, `tbhbdev version` names the bundle, a launch through the fallthrough works, and pointing the link back restores the old bundle | tools maintainer, ops maintainer |
| 5 | Wave 1: `tbhbdev agent` | The envelope corpus passes both ways in CI, and a Go driver launches on a phase 1 host | tools maintainer |
| 6 | Caller migration for each ported command | `rg 'tbhbdev alpha <command>'` finds nothing in tools, agent-config, ops, or wiki, and the forwarder logs stay empty for 14 days | agent-config maintainer, ops maintainer |
| 7 | Waves 2 and 3 | Each wave's commands answer under final names from the bundle | tools maintainer |
| 8 | The fallthrough retires, per D2 | `tbhbdev` with an unported name exits 127, and no bundle carries the Python wheel | tools maintainer |

Steps 1 and 2 run in parallel. Step 4 waits on D1 for signing, and step 8 waits on D2.

## Appendices

- [Decisions](decisions.md): 3 open, then the decided record.
- [A. gh architecture](A-gh-architecture.md): how cli/cli is built, and what this design copies or avoids.
- [B. gh pitfalls](B-gh-pitfalls.md): the architectural pain in cli/cli's history, by commit, each with the rule it gives this design.
- [C. Alternatives considered](C-alternatives.md): the options not taken, and why.
- [D. Evidence](D-evidence.md): each claim about tbhbdev and its neighbors, labeled existing, new, or unverified.
- [E. Plugin contract](E-plugin-contract.md): dispatch, the contract, the library surface, and the version transition.
- [F. Build and checks](F-build-and-checks.md): the Bazel layout, the disjoint inputs, and each check.
- [G. Names and migration](G-names-and-migration.md): the `alpha` name map and how callers move.
- [H. Waves](H-waves.md): what ports in which wave, and how the work runs in parallel.
