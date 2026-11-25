---
type: Design Draft
title: Port tbhbdev to Go as plugin binaries
description: How tbhbdev moves from Python to a Go front and per-command plugin binaries in one monorepo, with a Python fallthrough, final command names, the review tooling first, a cobra, koanf, and in-process CUE stack, a versioned protocol that never refuses a launch, and self-update at start.
tags: [tbhbdev, go, cli, plugins, bazel, design]
status: draft
owner: tools maintainer
updated: "2026-11-25"
decisions_open: 3
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words gazelle depguard golangci kubectl syscall buildinfo unported selfupdate libexec cobra pflag koanf cuelang gochecknoinits -->

# Port tbhbdev to Go as plugin binaries

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | tools maintainer | 2026-11-25 | 3 open decisions in [decisions.md](decisions.md), about 10 minutes |

## Summary

On 2026-11-25 the operator chose to port tbhbdev, about 37,700 lines of Python, to Go as plugin binaries in one monorepo, modeled on the GitHub command-line tool `gh`. This design gives the port's shape, stack, and wave order.

A small standard-library front, `tbhbdev`, execs `tbhbdev-<name>` from its release bundle, and hands every other name, `alpha` included, to the Python tbhbdev from the same commit. Plugins use cobra, pflag, koanf, and in-process Configure Unify Execute (CUE) validation, with explicit dependencies and no package state.

Wave 1 ports the unfinished review tooling, and the launcher follows. Drivers and hosts agree on a versioned schema, and tbhbdev updates itself at start without refusing a run.

Implementation waits for the operator's read of the merged design. Three questions remain open for the operator: release signing, what retires, and plugins outside the bundle.

## Proposal

When the first build merges, these parts exist in tbhb-dev/tools, beside the Python under `src/tbhbdev` (D8):

- **The front, `cmd/tbhbdev`** (new). A Go binary that maps argv to one route and replaces itself with that route's program. It does no network, config, or per-plugin work before the exec.
- **Plugin binaries, `cmd/tbhbdev-<name>`** (new). One per top-level command word, such as `tbhbdev-review` for `tbhbdev review post`. Each owns its subcommands and its own small dependency set.
- **The Go stack** (new). cobra and pflag for each plugin's commands, koanf for config, and `cuelang.org/go` for every schema check, each held to its packages by the rules in [appendix K](K-go-stack.md). The front uses none of them.
- **The Python fallthrough** (existing code, new place). The Python package from the same commit, installed inside the bundle, runs every name the front doesn't route to Go, including all of `tbhbdev alpha` (D5).
- **The release bundle** (new). One archive per platform, built from one commit, with the front, every plugin, the Python wheel, and a manifest. The tools continuous integration (CI) release job builds it, and the tools maintainer role runs that job.
- **The installer** (new behavior). It unpacks a bundle into its own versioned directory and swaps one symbolic link, so a failed install leaves the old bundle running. The ops maintainer role runs the first install on each host, and after that event-driven sync and self-update keep hosts current.
- **The protocol schema, `schemas/agent-protocol.json`** (new). One versioned file for the run request and the run metadata, with a CI job that fails when it changes without a version decision ([appendix I](I-protocol.md)).
- **The self-update helper, `cmd/tbhbdev-self-update`** (new). The input/output (I/O) side of self-update, run only when the front's pure decision says the build is behind or the cache is stale ([appendix J](J-self-update.md)).
- **A shared Go library** (new). Small packages for the plugin contract, routing, build stamps, terminal streams, process runs, config, and schemas. [Appendix E](E-plugin-contract.md) lists them.
- **Checks from the first commit** (new, on existing rules). The pull request size gate, an import-direction check that keeps I/O out of pure packages, ast-grep rules for I/O in unit tests, and test time caps. [Appendix F](F-build-and-checks.md) has the layout and each check.

Other roles carry parts of it. The rules maintainer adds the ast-grep test macro to tbhb-dev/rules, the agent-config maintainer moves hook and brief callers to final names, and the CI speed role owns CI wall time across repositories.

## How it works

### Dispatch

1. The front resolves its own path once, from `os.Executable` and `filepath.EvalSymlinks`, and reads the bundle manifest beside it.
2. It reads the self-update cache and calls the pure `selfupdate.Decide`. Most calls get `Run` and go on, and the rest exec the self-update helper first.
3. A pure function, `dispatch.Resolve`, maps argv and the manifest to one route. The order is front built-ins (`version` and `help`), then bundle plugins by the first command word, then the Python fallthrough.
4. The front sets the contract variables and calls `syscall.Exec`. The plugin inherits the process ID, the argv tail verbatim, the environment, and every open descriptor, so the forced command's standard input contract and the inherited push-token pipe survive as they are.
5. The plugin's own exit status is the process's exit status, since nothing waits on it. The front keeps 125, 126, and 127 for its own failures.

The front never searches `PATH` for plugins and never routes `alpha` to Go. Whether it may run plugins from outside the bundle is D3.

### The plugin contract

Each plugin gets the argv after its name, unparsed. It also gets `TBHBDEV_DISPATCHED=1`, `TBHBDEV_PROG`, `TBHBDEV_PATH`, `TBHBDEV_BUNDLE`, `TBHBDEV_BUNDLE_VERSION`, and `TBHBDEV_CONTRACT=1` beside the inherited environment. Every binary carries the same build stamp, linked from one package as awrel's `buildmeta` does, and prints it for a lone `--version`.

The manifest lists each plugin's name, file, and SHA-256 hash, the Python entry, and the contract version. The installer checks the hashes once, and the front trusts the manifest at run time. [Appendix E](E-plugin-contract.md) gives the full contract.

### The Go stack

The front and `dispatch.Resolve` use only the standard library. The front runs before any flag parsing and must pass unknown names and flags through untouched, which a flag library would consume or reject (D18).

Each plugin builds its command tree with cobra and pflag, wired to avoid gh's Factory and its global state. Each command's constructor takes a private struct of only the dependencies it uses, and no package holds state or registers commands in `init()`. koanf loads the config file once in `io/app` and hands pure code a plain struct.

Schemas are embedded with `go:embed` and validated in-process by `cuelang.org/go` in pure packages, and no Go code runs the `cue` binary (D20). The Python `src/tbhbdev/schemas/*.cue` files move over as the source, and [appendix K](K-go-stack.md) gives the depguard, go-arch-lint, and ast-grep rules that hold each library in place.

### Versions without refusal

Today the driver refuses any host whose build differs, through `same_version` in `decisions/driver.py`, and the request carries fixed versions that each reader matches exactly (existing). The port replaces both with one protocol version (D14).

- **One schema.** `schemas/agent-protocol.json` in tools defines the envelope, the run request, and the run metadata, with a top-level `version` that is the protocol version (D16). Python and Go validate against the same bytes. Writers are strict, and readers drop and log properties they don't know.
- **Two versions per build.** An additive change keeps the version. A breaking change bumps it, and each build serves its version and the previous one, as driver and as host. A bump merges only when every listed machine serves the version it replaces.
- **A version per launch.** Preflight reports the host's build and the versions it serves. The driver picks the highest common version and logs a build mismatch as a warning.
- **A CI check.** A planned job compares the schema at the merge base with the head, and fails unless `schemas/agent-protocol.decisions.toml` holds an entry for the new schema's hash with a version that matches its kind.

| Phase | Machines run | What keeps every launch going |
| --- | --- | --- |
| 1. The release that removes the refusal | Python, installed as today | It ships to machines that drive launches before machines that only host. A phase 1 driver warns on a build mismatch, and every host serves version 2 |
| 2. Bundle with Python behind the front | Phase 1 Python, or a bundle | Both sides choose a common version, and a build mismatch only logs |
| 3. Go `agent` in the bundle | Mixed Go and Python drivers and hosts | A Python driver reaches a Go host, and a Go driver a Python host, on a common version. Each build serves two versions, and skew stays within one |
| 4. Python server removed | Bundles only, forced command on Go | Versions change only under the schema rules |

A corpus of envelopes per served version tests every pairing in CI. [Appendix I](I-protocol.md#the-transition) walks each phase.

### Self-update

At start, the front compares its build's commit with a cached main commit (D15). The cache, `$XDG_CACHE_HOME/tbhbdev/main.json`, lives 10 minutes, so most starts read one small file and make no network call.

When the build is behind, the front execs the self-update helper. It downloads the bundle beside the install, checks its SHA-256 hash and provenance (D1), renames a whole versioned directory into place, and then renames a new link over `~/.local/bin/tbhbdev`, so there's always a `tbhbdev`. It then execs the new front with the same arguments.

- **Concurrent runs.** One lock lets one update run at a time, and a run that finds it taken runs its current build. Each run holds a shared lock on its own bundle until it exits, and pruning never removes a held bundle.
- **Failures.** No network, a bad hash, or a read-only install writes a warning and runs the current build, and the next try waits one cache lifetime.
- **Opt-out.** `TBHBDEV_NO_SELF_UPDATE=1`.
- **Sync.** Event-driven sync on merges to main, which the tools maintainer is building, runs the same install and writes the same cache. Sync is the main path, and self-update catches machines it missed.

### Pure packages and I/O packages

Pure packages take plain values and return data, as the Python `decisions/` package already does. I/O lives only under `io/` directories, in `cmd/`, and in each plugin's wiring package. No package takes the name `core` or `shell`.

These checks run from the first commit, and the two protocol checks from the phase 1 release, before any Go ships.

| Check | What it enforces | Status |
| --- | --- | --- |
| Pull request size gate | 400 added code lines and 1,500 raw lines per pull request, Go tests included | existing, rules `pr_size` |
| `go_arch_lint_test` | Pure packages import only pure packages, and `io` imports never point back | existing macro, new config |
| golangci-lint depguard | Pure files import only an allowlist of standard packages | existing aspect, new config |
| `ast_grep_test` | No file, process, network, clock, or environment calls in pure code or its tests | new macro in rules |
| Test caps | A unit test target fails after 10 seconds, and any Go test target after 5 minutes | new wrapper macros |
| Front budget | The front links only routing, contract, stamp, and self-update decision packages, under a size cap, and makes no network call | new |
| Stack rules | cobra, pflag, and koanf only in I/O wiring, `cuelang.org/go` only in pure schema packages, no `init()`, no `os/exec` outside one package, and no `cue` child process | existing linters, new config ([appendix K](K-go-stack.md#the-rules-that-hold-it)) |
| `protocol-version` CI job | A change to the protocol schema fails without a decision entry for its hash, an additive entry keeps the version, and a breaking one adds exactly 1 | new, in tools CI |
| Protocol corpus | Every writer emits only schema-valid documents, each side reads every served version's corpus, and the Python and Go schema copies match | new |
| Self-update | Table tests for every `Decide` row and cache-age boundary, two swaps at once, a prune beside a held bundle, and each failure running the current build | new |

### Bazel layout

The Go module is `go.tbhb.dev/tools` at the repository root, with rules_go, gazelle, and tbhb_rules_go set up as afakt and awrel set them up. Every Go package and the Python suite are separate Bazel packages with disjoint inputs, so a Go-only change leaves `pytest_test` unaffected in pull requests and in the merge queue (D7). [Appendix F](F-build-and-checks.md) has the tree and the input checks.

### Names and callers

Go has no `alpha`. Each command gets its final name when it ports, such as `tbhbdev agent exec` and `tbhbdev pr create`, and [appendix G](G-names-and-migration.md) maps every `alpha` command.

After a command ports, its Python `alpha` entry becomes a silent forwarder to the Go command and logs each call in the state directory. Hooks, briefs, scripts, and agent-config move to the final name, a check stops new `alpha` uses, and the Python code goes once the logs show no calls for 14 days.

### Port waves

Wave 1 ports the review tooling, the one unfinished part of tbhbdev: review runs, verdicts, posting reviews to GitHub, and the unbuilt pieces of the review-wire milestone plan (D17). Until wave 2, Go review code starts runs through `tbhbdev agent exec`, which the Python launcher answers under that final name.

Wave 2 ports the launcher, which works today. Wave 3 ports the harness hooks, the pull request commands, and `loop`, and wave 4 the rest. [Appendix H](H-waves.md) lists the review pieces in Python and the unbuilt ones, the line counts, and the slices: about five serial scaffolding pull requests, each read by the operator before merge (D19), then up to about eight at once.

### What gh taught

gh got the one-line `main`, the typed exit errors, and the injected process seams right ([appendix A](A-gh-architecture.md)). Its history shows what to avoid: plugins found on `PATH`, silent name collisions, a version stamp that silently didn't apply, five fixes to find its own path, and a 16-field Factory beside package state read in `init()`. [Appendix B](B-gh-pitfalls.md) ties each pitfall to a rule here, and [appendix K](K-go-stack.md) shows the cobra wiring that avoids the last two.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| Builds installed before phase 1 still refuse any other build | A launch from an old driver to a phase 1 host fails | Phase 1 installs on driving machines before hosts. Which machines drive is unverified, and step 2 lists them | existing: `same_version` |
| A machine falls two breaking versions behind | It shares no version with current drivers | A bump merges only when every listed machine serves the version it replaces, and sync and self-update move machines to main | new |
| A property marked additive isn't safe to ignore | An older host runs without a new limit | Such a property is breaking by rule, and the decision entry's reason is reviewed | new |
| A schema change lands without a version decision | Two builds read the same version differently | The `protocol-version` job, and strict writer validation that fails on any field the schema lacks | new |
| Self-update runs on a hook's start | A hook waits on a download or hits its timeout | Sync installs first, so updates at start are rare. A killed update leaves only a temporary directory, and the cache waits a lifetime before retrying | new |
| A bad bundle on main | Every machine takes it at its next start | Prior bundles stay on disk, a revert on main ships forward, and `TBHBDEV_NO_SELF_UPDATE=1` holds a machine | new |
| Concurrent runs during a swap | A run loses its files mid-run | Renames of whole directories and links, one update lock, and a shared lock per bundle that pruning respects | new |
| Go review code depends on the Python launcher until wave 2 | A review run fails if the Go call and the Python arguments drift | Python answers `tbhbdev agent` with the `alpha agent` code itself, and a test runs the Go review command against the Python launcher from the same commit | new |
| `cuelang.org/go` reads a JSON Schema differently from Python's `jsonschema` | The two languages accept different documents | The protocol corpus and the review fixtures run both validators on every valid and planted invalid document | unverified |
| `cuelang.org/go` adds start-up time, or reads its own environment | Plugins that validate start slower, or pure code depends on the environment | Only plugins that validate link it, never the front or `tbhbdev-hook`, and the schema slice measures both | unverified |
| The Go agent server behaves differently from the Python one | A launch fails or runs wrongly on a Go host | Both validate against the same schema bytes, the corpus runs both ways in CI, and the Go host hands requests it can't parse to Python | new |
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
| 2 | Phase 1 Python release: a warning in place of the build-version refusal, the protocol schema at version 2 with validation on both sides, `alpha agent protocol`, the `protocol-version` job, `TBHBDEV_PROG`, and `pytest_test` and the Python linters on Python inputs only. The release installs on driving machines first | A driver test with mismatched stamps launches, a schema edit with no decision entry fails the job, the release record lists the driving machines installed before any host, and a commit that adds only a `.go` file reports `//tests:pytest_test` as cached or not run | tools maintainer, ops maintainer |
| 3 | Go scaffolding: the front, `internal/dispatch`, `internal/plugin` with the cobra runner, `internal/buildinfo`, the koanf loader, `internal/schemas`, the Bazel setup, and every check in the table. Each pull request goes to the operator before it merges (D19) | `tbhbdev alpha agent status` through the front prints what the Python command prints, each check fails on its planted fixture, and each merged scaffolding pull request has the operator's review | tools maintainer |
| 4 | The bundle build, the release job, and the installer | On one host, `tbhbdev version` names the bundle, a launch through the fallthrough works, and pointing the link back restores the old bundle | tools maintainer, ops maintainer |
| 5 | Self-update: the release pointer, `internal/selfupdate`, and the self-update helper | On one host with a stale cache and an older bundle, `tbhbdev agent status` swaps to the main bundle and runs, and with no network it warns and runs the old one | tools maintainer |
| 6 | Wave 1: `tbhbdev review`, the review run's verdict and post, the Python `agent` name, and the unbuilt review pieces | One real review run posts its verdict as a GitHub review at the head it judged through the Go poster, and `tbhbdev review check N` reports what the Python command reports on the same pull request | tools maintainer |
| 7 | Wave 2: `tbhbdev agent` | The envelope corpus passes both ways in CI, and a Go driver launches on a phase 1 host | tools maintainer |
| 8 | Caller migration for each ported command | `rg 'tbhbdev alpha <command>'` finds nothing in tools, agent-config, ops, or wiki, and the forwarder logs stay empty for 14 days | agent-config maintainer, ops maintainer |
| 9 | Waves 3 and 4 | Each wave's commands answer under final names from the bundle | tools maintainer |
| 10 | The fallthrough retires, per D2 | `tbhbdev` with an unported name exits 127, and no bundle carries the Python wheel | tools maintainer |

No step starts until the operator has read the merged design (D17). Steps 1 and 2 then run in parallel, steps 4 and 5 wait on D1 for provenance, and step 10 waits on D2.

## Appendices

- [Decisions](decisions.md): 3 open, then the decided record.
- [A. gh architecture](A-gh-architecture.md): how cli/cli is built, and what this design copies or avoids.
- [B. gh pitfalls](B-gh-pitfalls.md): the architectural pain in cli/cli's history, by commit, each with the rule it gives this design.
- [C. Alternatives considered](C-alternatives.md): the options not taken, and why.
- [D. Evidence](D-evidence.md): each claim about tbhbdev and its neighbors, labeled existing, new, or unverified.
- [E. Plugin contract](E-plugin-contract.md): dispatch, the contract, the install layout, and the library surface.
- [F. Build and checks](F-build-and-checks.md): the Bazel layout, the disjoint inputs, and each check.
- [G. Names and migration](G-names-and-migration.md): the `alpha` name map and how callers move.
- [H. Waves](H-waves.md): what ports in which wave, the review tooling in Python and what's unbuilt, and how the work runs in parallel.
- [I. Protocol and versions](I-protocol.md): the versioned schema, its change rules and CI check, and the transition in which no launch is refused.
- [J. Self-update](J-self-update.md): the decision, the cache, the swap, failures, and how self-update relates to sync.
- [K. Go stack](K-go-stack.md): cobra, pflag, koanf, and `cuelang.org/go`, how commands get their dependencies, and the rules that keep each library in its place.
