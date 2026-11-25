---
type: Design Draft
title: Port tbhbdev to Go as plugin binaries
description: How tbhbdev moves from Python to a Go front and per-command plugin binaries in the tbhb-dev/tbhbdev monorepo, with a tbhbdev-py fallthrough, final command names, the review tooling first, a cobra, koanf, go-github, and in-process CUE stack, a versioned protocol that never refuses a launch, and self-update at start.
tags: [tbhbdev, go, cli, plugins, bazel, design]
status: draft
owner: tools maintainer
updated: "2026-11-25"
decisions_open: 3
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words gazelle depguard golangci kubectl syscall buildinfo unported selfupdate libexec cobra pflag koanf cuelang gochecknoinits ghapi -->

# Port tbhbdev to Go as plugin binaries

| Status | Owner | Updated | Needs from tbhb |
| --- | --- | --- | --- |
| Draft | tools maintainer | 2026-11-25 | 3 open decisions in [decisions.md](decisions.md), about 10 minutes |

## Summary

On 2026-11-25 tbhb chose to port tbhbdev, about 37,700 lines of Python, to Go as plugin binaries, modeled on the GitHub command-line tool `gh`. The Go code lives in the new monorepo tbhb-dev/tbhbdev (D22), and the Python stays in tbhb-dev/tools.

A small standard-library front, `tbhbdev`, execs `tbhbdev-<name>` from its release bundle, and hands every other name, `alpha` included, to `tbhbdev-py`, the Python tbhbdev beside it (D25). Plugins use cobra, koanf, go-github, and in-process Configure Unify Execute (CUE) validation, with no package state.

Wave 1 ports the unfinished review tooling, and the launcher follows. Drivers and hosts agree on a versioned schema, and tbhbdev updates itself without blocking a run.

Implementation waits for tbhb's read of the merged design.

## Proposal

When the first build merges, these parts exist. The Go code is one module in tbhb-dev/tbhbdev, `github.com/tbhb-dev/tbhbdev`, a path written only in its `go.mod` (D22). The Python tbhbdev, `tbhbdev-py`, and every unported command stay in tbhb-dev/tools.

- **The front, `factory/tbhbdev/cmd/tbhbdev`** (new). A Go binary that maps argv to one route and replaces itself with that route's program. It does no network, config, or per-plugin work before the exec.
- **Plugin binaries, `factory/tbhbdev/cmd/tbhbdev-<name>`** (new). One per top-level command word, such as `tbhbdev-review` for `tbhbdev review post`. Each owns its subcommands and its own small dependency set.
- **The Go stack** (new). cobra and pflag for each plugin's commands, koanf for config, `cuelang.org/go` for every schema check, and go-github behind one package for GitHub, each held to its packages by the rules in [appendix K](K-go-stack.md). The front uses none of them.
- **The Python fallthrough, `tbhbdev-py`** (existing code, new place). The Python wheel from the tools commit that the monorepo pins, installed inside the bundle, with its `tbhbdev-py` entry point in the front's own `bin/` folder. It runs every name the front does not route to Go, including all of `tbhbdev alpha` (D5, D25).
- **The release bundle** (new). One archive per platform, built by the tbhb-dev/tbhbdev release job from one monorepo commit: the front, every plugin, the pinned tools wheel, and a manifest that names both commits. tbhb-agent builds the code, tbhb approves its merge, and the release job then builds the bundle from the merged commit.
- **Release checks** (new, D23). The release job attests each archive with a keyless GitHub artifact attestation. The installer checks the SHA-256 hashes and the attestation, pinned to the tbhbdev release workflow, the `main` ref, and the expected commit, before any swap.
- **The installer** (new behavior). It unpacks a bundle into its own versioned directory and swaps one symbolic link, so a failed install leaves the old bundle running. tbhb-agent runs the first install on each host, a deploy kept apart from tbhb's approval of the code, and after that event-driven sync and self-update keep hosts current.
- **The protocol schema, `schemas/agent-protocol.json` in tools** (new). One versioned file for the run request and the run metadata, with a tools CI job that fails when it changes without a version decision ([appendix I](I-protocol.md)). The monorepo embeds a byte-checked copy.
- **The self-update helper, `factory/tbhbdev/cmd/tbhbdev-self-update`** (new). The input/output (I/O) side of self-update, run only when the front's pure decision says the build is behind or the cache is stale, inside a fixed time budget ([appendix J](J-self-update.md)).
- **A shared Go library** (new). Small packages for the plugin contract, routing, build stamps, terminal streams, process runs, config, schemas, and the GitHub client. [Appendix E](E-plugin-contract.md) lists them.
- **Checks from the first commit** (new, on existing rules). The pull request size gate, an import-direction check that keeps I/O out of pure packages, ast-grep rules for I/O in unit tests, and test time caps. [Appendix F](F-build-and-checks.md) has the layout and each check.

Parts of it land in other repositories, with the same duties kept apart. tbhb-agent builds the ast-grep test macro in tbhb-dev/rules and moves hook and brief callers to final names in agent-config, a reviewer run on a different provider reviews each pull request, and tbhb approves it. CI wall time across repositories is separate CI speed work that tbhb-agent tracks.

### The CLI folder

The CLI sits in `factory/tbhbdev` inside tbhb-dev/tbhbdev, decided by tbhb (D27). `factory/` is flat, with no grouping folders such as `factory/cli`. Every Go path in this design, such as `cmd/tbhbdev` or `internal/dispatch`, sits under `factory/tbhbdev`, so its packages are `github.com/tbhb-dev/tbhbdev/factory/tbhbdev/...`, and the module path stays the one line in `go.mod`.

## How it works

### Dispatch

1. The front resolves its own path once, from `os.Executable` and `filepath.EvalSymlinks`, and takes a shared lease on that bundle before it reads anything in it.
2. It reads the bundle manifest beside it and the self-update cache, and calls the pure `selfupdate.Decide`. Most calls get `Run` and go on, and the rest exec the self-update helper first. If that exec fails, the front goes on to dispatch the current build.
3. A pure function, `dispatch.Resolve`, maps argv and the manifest to one route. The order is front built-ins (`version` and `help`), then bundle plugins by the first command word, then `tbhbdev-py`.
4. The front sets the contract variables and calls `syscall.Exec`. The plugin inherits the process ID, the argv tail verbatim, the environment, and every open descriptor, so the forced command's standard input contract and the inherited push-token pipe survive as they are.
5. The plugin's own exit status is the process's exit status, since nothing waits on it. The front keeps 125, 126, and 127 for its own failures.

The front never searches `PATH` for plugins or for Python, and never routes `alpha` to Go. Whether it may run plugins from outside the bundle is D3.

### The plugin contract

Each plugin gets the argv after its name, unparsed. It also gets `TBHBDEV_DISPATCHED=1`, `TBHBDEV_PROG`, `TBHBDEV_PATH`, `TBHBDEV_BUNDLE`, `TBHBDEV_BUNDLE_VERSION`, and `TBHBDEV_CONTRACT=1` beside the inherited environment. Every binary carries the same build stamp, linked from one package as awrel's `buildmeta` does, and prints it for a lone `--version`.

The manifest lists each plugin's name, file, and SHA-256 hash, the `tbhbdev-py` entry with the tools commit it came from, and the contract version. The installer checks the hashes once, and the front trusts the manifest at run time. [Appendix E](E-plugin-contract.md) gives the full contract.

### The Go stack

The front and `dispatch.Resolve` use only the standard library. The front runs before any flag parsing and must pass unknown names and flags through untouched, which a flag library would consume or reject (D18).

Each plugin builds its command tree with cobra and pflag, wired to avoid gh's Factory and its global state. Each command's constructor takes a private struct of only the dependencies it uses, and no package holds state or registers commands in `init()`. koanf loads the config file once in `io/app` and hands pure code a plain struct.

Schemas are embedded with `go:embed` and validated in-process by `cuelang.org/go` in pure packages, and no Go code runs a `cue` binary by any path (D20). The I/O shell clears the environment variables that CUE reads before the first schema call.

GitHub calls go through go-github, pinned at one major, which only `internal/ghapi` imports. A small GraphQL helper on the same client covers what REST lacks, and app installation tokens come from a standard-library JSON Web Token (JWT) with go-github's installation calls (D24). [Appendix K](K-go-stack.md) gives the rules that hold each library in place.

### Versions without refusal

Today the driver refuses any host whose build differs, through `same_version` in `decisions/driver.py`, and the request carries fixed versions that each reader matches exactly (existing). The port replaces both with one protocol version (D14).

- **One schema.** `schemas/agent-protocol.json` in tools defines the envelope, the run request, and the run metadata, with a top-level `version` that is the protocol version (D16). Python and Go validate against the same bytes. Writers are strict, and readers drop and log properties they do not know.
- **Two versions per build.** An additive change keeps the version. A breaking change bumps it, and each build serves its version and the previous one, as driver and as host. A bump merges only when every listed machine serves the version it replaces.
- **A version per launch.** Preflight reports the host's build and the versions it serves. The driver picks the highest common version, writes the request and the staged request file in that version's shape, and logs a build mismatch as a warning. A host that cannot answer gets the legacy version 2 shape.
- **Bridges.** Every install keeps the newest bundle that serves each version from 2 up. When skew is wider than one and an update of the older side fails, the launch runs through a bridge on whichever side lacks the version.
- **Served and retained schemas.** Each build serves two versions for new launches and retains every frozen schema for reading runs already on disk. CI checks each set apart.
- **Run files carry their version.** The host records the chosen version as each run's first file, and every later reader validates the run's files against that version, across upgrades.
- **A CI check.** A planned tools job compares the schema at the merge base with the head, and fails unless `schemas/agent-protocol.decisions.toml` holds an entry for the new schema's hash with a version that matches its kind.

| Phase | Machines run | What keeps every launch going |
| --- | --- | --- |
| 1. The release that removes the refusal | Python, installed as today | A rollout barrier: launches pause, the ops maintainer syncs every listed machine to the phase 1 release and fences off any machine the sync cannot reach, and launches resume only when no listed machine runs an older build |
| 2. Bundle with `tbhbdev-py` behind the front | Phase 1 Python, or a bundle | Both sides choose a common version, and a build mismatch only logs |
| 3. Go `agent` in the bundle | Mixed Go and Python drivers and hosts | A Python driver reaches a Go host, and a Go driver a Python host, on a common version. Each build serves two versions, skew stays within one, and bridges cover any wider skew |
| 4. Python server removed | Bundles only, forced command on Go | Versions change only under the schema rules |

A corpus of envelopes per served version tests every pairing in CI. [Appendix I](I-protocol.md#the-transition) walks each phase.

### Self-update

At start, the front compares its build's release number, the count of first-parent commits on main, with a cached main release (D15). The cache, `$XDG_CACHE_HOME/tbhbdev/main.json`, lives 10 minutes, so most starts read one small file and make no network call.

When the build is behind, the front execs the self-update helper. It works inside a 3-second budget. Within it, the helper downloads the bundle, checks it (D23), renames a whole versioned directory into place, runs the new front once to prove it works, swaps the link, and execs the new front while still holding the update lock. Past it, a detached installer finishes the work and the current build runs at once.

- **Concurrent runs.** One lock lets one update run at a time, and a run that finds it taken runs its current build. Each run holds a shared lease on its own bundle until it exits, and pruning never removes a leased bundle.
- **Failures.** No failure blocks dispatch. No network, a bad hash, a failed exec of the helper, or a read-only install writes a warning and runs the current build, and a failed handoff puts the old link back.
- **Opt-out.** `TBHBDEV_NO_SELF_UPDATE=1`.
- **Sync.** Event-driven sync on merges to the monorepo's main, which the tools maintainer is building, runs the same install and writes the same cache. Sync is the main path, and self-update catches machines it missed.

### Pure packages and I/O packages

Pure packages take plain values and return data, as the Python `decisions/` package already does. I/O lives only under `io/` directories, in `cmd/`, and in each plugin's wiring package. No package takes the name `core` or `shell`.

These checks run from the first monorepo commit under `factory/tbhbdev`, and the two protocol checks from the phase 1 release in tools, before any Go ships.

| Check | What it enforces | Status |
| --- | --- | --- |
| Pull request size gate | 400 added code lines and 1,500 raw lines per pull request, Go tests included | existing, rules `pr_size` |
| tbhb gate paths | A pull request that changes `factory/tbhbdev/cmd/tbhbdev`, `factory/tbhbdev/internal/dispatch/**`, or `factory/tbhbdev/internal/plugin/**` gets the `tbhb-gate` label, so coordination puts it on Open after review and CI and nobody approves it before tbhb's go (D19). No other path gets the label | new, in monorepo CI |
| `go_arch_lint_test` | Pure packages import only pure packages, and `io` imports never point back | existing macro, new config |
| golangci-lint depguard | Pure files import only an allowlist of standard packages | existing aspect, new config |
| `ast_grep_test` | No file, process, network, clock, or environment calls in pure code or its tests | new macro in rules |
| Test caps | A unit test target fails after 10 seconds, and any Go test target after 5 minutes | new wrapper macros |
| Front budget | The front links only routing, contract, stamp, and self-update decision packages, under a size cap, and makes no network call | new |
| Stack rules | cobra, pflag, and koanf only in I/O wiring, `cuelang.org/go` only in pure schema packages, go-github only in `internal/ghapi`, no `init()`, no `os/exec` outside one package, and no `cue` child process by name or path | existing linters, new config ([appendix K](K-go-stack.md#the-rules-that-hold-it)) |
| CUE boundary | The CUE variables are cleared before the first CUE call, a CUE bump that adds an environment read to validation or to linked task code fails, and no task runner is linked | new |
| `protocol-version` CI job | A change to the protocol schema fails without a decision entry for its hash, an additive entry keeps the version, a breaking one adds exactly 1 and freezes the base, and no retained schema changes or goes | new, in tools CI |
| Protocol corpus | Every writer emits only schema-valid documents, each side reads every served version's corpus and the legacy shape, every reader reads one run per retained version, launches through a bridge work on either side, and the tools and monorepo schema copies match | new |
| Self-update | Table tests for every `Decide` row and the release order, the budget, each failure falling through to dispatch, the proof before publish, link restore under the held lock, publication under concurrent callers, lease order, and no older publish | new |

### Bazel layout

The monorepo tbhb-dev/tbhbdev holds one Go module at its root, with rules_go, gazelle, and tbhb_rules_go set up as afakt and awrel set them up. Every Go package is its own Bazel package under `factory/tbhbdev`, so a pull request tests only what it affects (D7). tools keeps the Python suite and never runs on a monorepo change. [Appendix F](F-build-and-checks.md) has the tree.

### Names and callers

Go has no `alpha`. Each command gets its final name when it ports, such as `tbhbdev agent exec` and `tbhbdev pr create`, and [appendix G](G-names-and-migration.md) maps every `alpha` command.

After a command ports, its Python `alpha` entry becomes a silent forwarder to the Go command and logs each call in the state directory. Hooks, briefs, scripts, and agent-config move to the final name, a check stops new `alpha` uses, and the Python code goes once the logs show no calls for 14 days.

### Port waves

Wave 1 ports the review tooling, the one unfinished part of tbhbdev: review runs, verdicts, posting reviews to GitHub, and the unbuilt pieces of the review-wire milestone plan (D17). It keeps the review authority of ADR 0004 and ADR 0009: GO or HOLD verdicts, comment reviews on the exact judged head, and approval left to the independent approver. Until wave 2, Go review code starts runs through `tbhbdev agent exec`, which the Python launcher answers under that final name.

Wave 2 ports the launcher, which works today. Wave 3 ports the harness hooks, the pull request commands, and `loop`, and wave 4 the rest. [Appendix H](H-waves.md) lists the review pieces in Python and the unbuilt ones, the line counts, and the slices.

The scaffolding is about six serial pull requests in tbhb-dev/tbhbdev, then up to about eight at once. The first Go slice, tools#418, waits for the move to the monorepo and does not merge into tools.

### The scaffolding gate

A pull request that changes the front, dispatch, or plugin plumbing, `factory/tbhbdev/cmd/tbhbdev`, `factory/tbhbdev/internal/dispatch/**`, or `factory/tbhbdev/internal/plugin/**`, goes to tbhb before it merges (D19). Review and CI come first, and coordination then puts the pull request on Open. Nobody approves it or runs `tbhb-approve` on it until tbhb says go, commissioned runs included, and a reviewer's verdict on one is a GO or HOLD comment only.

The monorepo's general scaffolding is not gated, and neither is any other pull request under `factory/tbhbdev` that leaves those paths alone.

### What gh taught

gh got the one-line `main`, the typed exit errors, and the injected process seams right ([appendix A](A-gh-architecture.md)). Its history shows what to avoid: plugins found on `PATH`, silent name collisions, a version stamp that silently did not apply, five fixes to find its own path, and a 17-field Factory beside package state read in `init()`. [Appendix B](B-gh-pitfalls.md) ties each pitfall to a rule here, and [appendix K](K-go-stack.md) shows the cobra wiring that avoids the last two.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| Builds installed before phase 1 still refuse any other build, and tools#400's sync-once retry still demands an exact match | A launch between an old and a phase 1 machine fails | The rollout barrier in plan step 2: launches pause until every listed machine runs phase 1 or later. A machine the sync cannot reach is fenced off the list, with its driver key removed from every host, and rejoins only after a sync and a version check | existing: `same_version` |
| A newer v2 writer reaches a host from before phase 1 | That host refuses the unknown fields | A host that cannot answer the protocol query gets the frozen legacy v2 shape, in the envelope and the staged request file | existing: `decisions/request.py` |
| A machine falls two breaking versions behind | It shares no version of its own with current drivers | A bump merges only when every listed machine serves the version it replaces. The driver updates the lagging side once, and if that fails the launch runs through a bridge bundle that serves the older version, with a warning | new |
| A property marked additive is not safe to ignore | An older host runs without a new limit | Such a property is breaking by rule, and the decision entry's reason is reviewed | new |
| A schema change lands without a version decision | Two builds read the same version differently | The `protocol-version` job, and strict writer validation that fails on any field the schema lacks | new |
| Self-update runs on a hook's start | A hook waits on a download or hits its timeout | A 3-second budget, after which a detached installer finishes the work with no inherited descriptors | new |
| A bad bundle on main | Every machine takes it at its next start | Prior bundles stay on disk, a revert on main ships forward, and `TBHBDEV_NO_SELF_UPDATE=1` holds a machine | new |
| Concurrent runs during a swap | A run loses its files mid-run, a slow update publishes an older bundle, or a caller hits a bundle that cannot run | A lease taken before any bundle read, held across the re-exec, one update lock held from the recheck until the new front starts, a proof run of the new bundle before the link moves, and release numbers in place of commit times | new |
| Go review code depends on the Python launcher until wave 2 | A review run fails if the Go call and the Python arguments drift | Python answers `tbhbdev agent` with the `alpha agent` code itself, and a test runs the Go review command against the pinned Python launcher | new |
| tools main gains review changes after tools#399 merged | Wave 1 ports a poster that does not match main | Wave 1 re-reads tools main on the day it starts, and its contract tests include the submitted-commit check that tools#399 added | existing: tools#399 merged as `719b589` |
| The Python and Go copies of a schema drift across two repositories | The two languages accept different documents | A monorepo test compares each embedded copy with the pinned tools commit byte for byte | new |
| `cuelang.org/go` reads a JSON Schema differently from Python's `jsonschema` | The two languages accept different documents | The protocol corpus and the review fixtures run both validators on every valid and planted invalid document | unverified |
| `cuelang.org/go` adds start-up time | Plugins that validate start slower | Only plugins that validate link it, never the front or `tbhbdev-hook`, and the schema slice measures it | unverified |
| The Go agent server behaves differently from the Python one | A launch fails or runs wrongly on a Go host | Both validate against the same schema bytes, the corpus runs both ways in CI, and the Go host hands requests it cannot parse to `tbhbdev-py` | new |
| The forced command or the push-token descriptor does not survive exec | Remote launches break | An integration test per host operating system, in the first build | unverified |
| Timeout and kill behavior drifts from `process_group.py` | Runs hang or die early | Port the Python tests for it first, as Go table tests | unverified |
| Go code runs longer than the Python it replaces | Slices split smaller under the 400-line gate | Size slices by Go lines after the first two | unverified |
| A gap in the depguard or ast-grep rules | I/O slips into a pure test | Each rule has a planted failing fixture | new |
| State files read by both languages drift | One side misreads the other's state | Go reads and writes the Python formats, with shared fixtures | new |
| A Python fix in tools waits for a pin bump in the monorepo | A tools merge reaches machines later than today | Each tools merge to main opens a pin bump in tbhb-dev/tbhbdev, which sync ships like any other merge | new |
| Library callers in other repositories | Removing Python modules breaks them | They stay in Python until their callers move | unverified |

[Appendix D](D-evidence.md) lists the evidence behind each claim.

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | `ast_grep_test` in tbhb-dev/rules | A rules test fails on a planted `os.ReadFile` in a pure test fixture | rules maintainer |
| 2 | Phase 1 Python release in tools: a warning in place of the build-version refusal and of tools#400's exact-match retry, the protocol schema at version 2 with validation on both sides, the frozen legacy v2 shape, the top-level Python `agent` name with `agent protocol`, the version file for each run, the `protocol-version` job, `TBHBDEV_PROG`, and the `tbhbdev-py` entry point. Then the rollout barrier: launches pause, the ops maintainer syncs every listed machine, and `tbhbdev alpha version-check` shows none older | A driver test with mismatched stamps launches, a schema edit with no decision entry fails the job, and the barrier record lists every listed machine on phase 1 or later, with any unreachable machine fenced off the list, before launches resume | tools maintainer, ops maintainer |
| 3 | Go scaffolding in tbhb-dev/tbhbdev under `factory/tbhbdev` (D27): the front, `internal/dispatch`, `internal/plugin` with the cobra runner, `internal/buildinfo`, the koanf loader, `internal/schemas`, `internal/ghapi`, the Bazel setup, and every check in the table. tools#418 moves here. The two scaffolding pull requests that change the front, dispatch, or plugin plumbing wait for tbhb's go (D19) | `tbhbdev alpha agent status` through the front prints what the Python command prints, each check fails on its planted fixture, and each gated pull request has tbhb's go | tools maintainer |
| 4 | The bundle build, the release job with its attestation, and the installer | On one host, `tbhbdev version` names the bundle, a launch through the fallthrough works, a bundle with a wrong workflow, ref, or commit in its attestation is refused, and pointing the link back restores the old bundle | tools maintainer, ops maintainer |
| 5 | Self-update: the release pointer, `internal/selfupdate`, and the self-update helper | On one host with a stale cache and an older bundle, `tbhbdev alpha agent status` swaps to the main bundle and runs, with no network it warns and runs the old one, and with a slow download it runs the old one within the budget | tools maintainer |
| 6 | Wave 1: `tbhbdev review`, the review run's verdict and post, and the unbuilt review pieces | One real review run posts a GO or HOLD comment review at the head it judged through the Go poster with the host-held token, and `tbhbdev review check N` reports what the Python command reports on the same pull request | tools maintainer |
| 7 | Wave 2: `tbhbdev agent` | The envelope corpus passes both ways in CI, a Go driver launches on a phase 1 host, and `tbhbdev agent env` reaches the Python code through the plugin's passthrough | tools maintainer |
| 8 | Caller migration for each ported command | `rg 'tbhbdev alpha <command>'` finds nothing in tools, tbhbdev, agent-config, ops, or wiki, and the forwarder logs stay empty for 14 days | agent-config maintainer, ops maintainer |
| 9 | Waves 3 and 4 | Each wave's commands answer under final names from the bundle | tools maintainer |
| 10 | The fallthrough retires, per D2 | `tbhbdev` with an unported name exits 127, and no bundle carries the Python wheel | tools maintainer |

No step starts until tbhb has read the merged design (D17). Steps 1, 2, and 3 then run in parallel, and step 3's ast-grep check lands once step 1 has. Until wave 4 ports `install check`, every fleet check uses the existing `tbhbdev alpha version-check`, which step 2 extends to print each machine's served versions. Step 10 waits on D2.

## Appendices

- [Decisions](decisions.md): 3 open, then the decided record.
- [A. gh architecture](A-gh-architecture.md): how cli/cli is built, and what this design copies or avoids.
- [B. gh pitfalls](B-gh-pitfalls.md): the architectural pain in cli/cli's history, by commit, each with the rule it gives this design.
- [C. Alternatives considered](C-alternatives.md): the options not taken, and why.
- [D. Evidence](D-evidence.md): each claim about tbhbdev and its neighbors, labeled existing, new, or unverified.
- [E. Plugin contract](E-plugin-contract.md): dispatch, the contract, the install layout, and the library surface.
- [F. Build and checks](F-build-and-checks.md): the Bazel layout and each check.
- [G. Names and migration](G-names-and-migration.md): the `alpha` name map and how callers move.
- [H. Waves](H-waves.md): what ports in which wave, the review tooling in Python and what is unbuilt, and how the work runs in parallel.
- [I. Protocol and versions](I-protocol.md): the versioned schema, its change rules and CI check, and the transition in which no launch is refused.
- [J. Self-update](J-self-update.md): the decision, the cache, the budget, the swap, failures, and how self-update relates to sync.
- [K. Go stack](K-go-stack.md): cobra, pflag, koanf, `cuelang.org/go`, and go-github, how commands get their dependencies, and the rules that keep each library in its place.
