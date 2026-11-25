---
type: Design Appendix
title: tbhbdev Go port decisions
description: The open and decided questions of the tbhbdev Go port design.
tags: [tbhbdev, go, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words gazelle depguard ITSM syscall unported libexec cobra pflag koanf cuelang cuecontext ghapi ghinstallation githubv -->

# tbhbdev Go port decisions

Back to [the design](README.md).

## Open

### D1. Is the keyless attestation enough, with no signing key held by tbhb-dev, and do bundles ship as GitHub Releases?

- **Recommendation:** yes to both.
- **Why:** D23 already decides the baseline: a keyless GitHub artifact attestation on each archive and SHA-256 checks, with the installer pinning the attestation to the tbhbdev release workflow, the `main` ref, and the expected commit. What stays open is whether tbhb-dev also holds a key, and where bundles publish. Self-update (D15) installs a new bundle on its own at start, so the check must need nothing an unattended run lacks, and an attestation needs no key to keep or rotate. Neither afakt nor awrel signs or attests today (unverified beyond the outline's grep).
- **If yes:** the release job publishes every main commit's bundle and the `main.json` pointer as GitHub Release assets of tbhb-dev/tbhbdev, and D23's checks are the whole trust check.
- **If no:** the operator names a held key or another channel. The admin VM role creates the key in 1Password, the release job signs with it beside the attestation, and every bundle carries the public key that its self-update checks the next bundle with, which adds a rotation runbook.
- **Reversible:** yes. A later release can add a held signature beside the attestation, since the installer checks whatever the manifest declares.

### D2. Does `tbhbdev host` retire without a port, and does `tbhbdev analytics` leave the bundle as its own Python tool, so the Python fallthrough can end?

- **Recommendation:** yes.
- **Why:** `host` is 2,395 lines that a memory note calls frozen until ops replaces it (unverified as current). `analytics` is 727 lines of PostgreSQL migrations with no tie to the launcher. Porting either costs more than it returns, and keeping either in the bundle keeps a Python interpreter in every install for good. Neither speaks the agent protocol, so neither touches D14 or D16.
- **If yes:** `host` stays in the fallthrough until ops ships its replacement, then goes. `analytics` moves to its own package with its own console script before plan step 9. The library-only modules stay in Python until their callers in other repositories move.
- **If no:** `host` or `analytics` joins wave 4 as Go, about 2,400 or 700 more Python lines to port, or the bundle keeps the Python wheel with no end date.
- **Reversible:** yes until the Python code is deleted, then only by restoring it from history.

### D3. May the front run plugins from outside its release bundle, but only from absolute paths listed in an operator-owned file, and never from a `PATH` search?

- **Recommendation:** yes.
- **Why:** graph and the private ITSM repository add tools today through Python entry points in the `tbhbdev.plugins` group, and they need a way in once the fallthrough ends. Agent hooks run `tbhbdev` on every tool call, so what it may run is a trust boundary. gh dropped its `PATH` search six days after it shipped ([appendix B](B-gh-pitfalls.md#11-path-discovery-dropped-after-six-days)).
- **If yes:** the front reads `$XDG_CONFIG_HOME/tbhbdev/plugins.toml` only when no bundle plugin matches. Each entry is a name and an absolute path, and a name that a built-in or bundle plugin uses is reported, never run. Self-update replaces only the bundle, so listed plugins keep their own installs, and one whose contract version differs runs with a logged warning (D14).
- **If no:** only the bundle's plugins run. graph and the ITSM repository ship their own commands under their own names once the fallthrough ends.
- **Reversible:** yes. Removing the file read narrows the boundary again, and the entries just stop working.

## Decided

### D26. The `agent` plugin owns its word and passes unported subcommands to `tbhbdev-py`

- **Decided:** 2026-11-25, by the tools maintainer role, under D10 and D25.
- **Rule:** the wave 2 `tbhbdev-agent` plugin owns the word `agent`. `agent env` and `agent gh-shim` port in wave 3, so until then the plugin's own passthrough execs `<bundle>/bin/tbhbdev-py agent env` or `agent gh-shim` with the rest of argv, and Python's top-level `agent` answers them with the `alpha agent-env` and `alpha gh-shim` code. The passthrough list comes from `names.toml` and empties in wave 3. Passthrough counts as covering a subcommand for the collision rule in [appendix G](G-names-and-migration.md#collisions-with-python-names).
- **Why this over moving them to wave 2:** `gh-shim` runs on the hook path and shares the wave 3 identity and guard code, so moving it would pull hook code into the launcher wave. The passthrough keeps wave 2's scope and leaves no `agent` subcommand without an answer.
- **Undo:** port both in wave 2 and drop the passthrough.
- **Record:** this design's pull request.

### D25. The Python fallthrough is `tbhbdev-py` in the front's own folder

- **Decided:** 2026-11-25, by the operator.
- **Rule:** the tools wheel gets a second entry point, `tbhbdev-py`, with the same code as `tbhbdev`. The installer puts it in the bundle's `bin/` beside the front, and the front execs it by absolute path. It never runs `python3 -m tbhbdev` and never searches `PATH`.
- **Undo:** a new operator decision.
- **Record:** the operator's decision of 2026-11-25 on the fallthrough.

### D24. The GitHub client stack is go-github behind `internal/ghapi`

- **Decided:** 2026-11-25, by the operator, on a research report into Go GitHub clients.
- **Rule:** REST goes through `github.com/google/go-github`, pinned at one major, v93.0.0 today, and only packages under `<cli-root>/internal/ghapi` import it. It builds the one client with an injected transport, token, User-Agent, and timeout, through only the constructor options that exist in both v93 and go-github's main branch: `WithTransport`, `WithAuthToken`, `WithUserAgent`, and `WithTimeout`. A small GraphQL helper on the same client covers what REST lacks: marking a pull request ready, the merge queue state query and enqueue, and Projects v2 by node ID. It doesn't use `githubv4`. App installation tokens come from a standard-library JWT and go-github's installation calls, not `ghinstallation`, which pins go-github v88. The token code keeps today's lookup of the repository installation and then the organization one, its cache, its cooldown, and its secret-free errors. go-github majors move on tbhb-dev's schedule, since six of the eight majors from v86 to v93 broke calls this code uses, all renames or by-value changes that the compiler finds. go-arch-lint and depguard allow go-github only in `internal/ghapi` ([appendix K](K-go-stack.md#github-with-go-github)).
- **Undo:** a new operator decision. A swap touches only `internal/ghapi`.
- **Record:** the operator's decisions of 2026-11-25 on the GitHub client stack.

### D23. Bundles are checked by keyless attestation and SHA-256, pinned to workflow, ref, and commit

- **Decided:** 2026-11-25, by the tools maintainer role, as this design's baseline. D1 asks only whether to add a held key and where bundles publish.
- **Rule:** the tbhb-dev/tbhbdev release job attests each archive with `actions/attest-build-provenance`, a keyless GitHub artifact attestation. Before any swap, the installer, sync, and self-update check the archive's SHA-256 hash against the pointer and every file hash against the manifest. They then verify the attestation with the signer workflow pinned to the tbhbdev release workflow, the source ref pinned to `refs/heads/main`, and the source commit pinned to the pointer's commit, which must equal the manifest's commit. A bundle whose commit is older than the installed one is never published. Any failed check warns and keeps the current build (D15).
- **Undo:** a new design decision. Dropping a pin widens what a host trusts.
- **Record:** this design's pull request.

### D22. The Go code lives in the monorepo tbhb-dev/tbhbdev

- **Decided:** 2026-11-25, by the operator, superseding D8.
- **Rule:** the Go port is built in tbhb-dev/tbhbdev as one module, `github.com/tbhb-dev/tbhbdev`. The module path appears only in `go.mod`, so an organization rename touches one place. The CLI sits in the folder `<cli-root>`, which stays open with two candidates ([the design](README.md#the-open-cli-folder)). tools stays the home of the Python `tbhbdev-py` and every unported command. One monorepo commit builds the bundle: the Go binaries from that commit, and the wheel from the tools commit it pins, with both commits in the manifest. The first Go slice, tools#418, waits for the move and doesn't merge into tools.
- **Undo:** a new operator decision.
- **Record:** the operator's decision of 2026-11-25 on the monorepo.

### D21. The review commands take the top-level name `review`

- **Decided:** 2026-11-25, by the tools maintainer role, under D17 and refining D10.
- **Rule:** `alpha review draft`, `check`, `policy`, `post`, `dismiss`, and `reply` port as `tbhbdev review` in their own plugin, `tbhbdev-review`, and not as `pr review` in `tbhbdev-pr`. Wave 1 ships the review commands alone. A `tbhbdev-pr` that answered only `review` would send every other `pr` word to a Go usage error until wave 3.
- **Undo:** rename before callers move. After they move, a rename is a second migration.
- **Record:** this design's pull request.

### D20. Go validates schemas in-process with `cuelang.org/go`

- **Decided:** 2026-11-25, by the operator.
- **Rule:** the Go port never runs a `cue` binary, by name or by any path. Every schema check goes through `cuelang.org/go` in a pure package, with the schema embedded by `go:embed`, including the issue and pull request templates and the D16 run request and run metadata schema. depguard and ast-grep rules forbid running `cue` and forbid `os/exec` in pure packages, and the one process constructor refuses any program whose base name is `cue`. `cuecontext.New` reads `CUE_DEBUG` and `CUE_EXPERIMENT`, so the I/O shell clears every `CUE_` variable before the first CUE call, and a test fails when a CUE upgrade adds an environment read ([appendix K](K-go-stack.md#the-cue-boundary)). The tools `schemas/*.cue` files stay the source while Python reads them, and the monorepo embeds byte-checked copies from the pinned tools commit.
- **Undo:** a new operator decision.
- **Record:** the operator's decisions of 2026-11-25 on the wave order and the stack, relayed with D17 to D19.

### D19. Scaffolding pull requests go to the operator before merge

- **Decided:** 2026-11-25, by the operator.
- **Rule:** each Go scaffolding pull request goes to the operator before it merges. These are the serial pull requests in [appendix H](H-waves.md#go-scaffolding) that set up the front, the plugin runner, the stack, the schema package, and the checks.
- **The gate, refined the same day:** pull requests for the front, dispatch, and plugin plumbing don't merge, and nobody approves them or runs `tbhb-approve` on them, until the operator says go on Open. That covers every session and subagent, commissioned runs included. Review and CI come first, and coordination then puts the pull request on Open. A reviewer's verdict on one is a GO or HOLD comment only.
- **In the monorepo:** the gate carries over to tbhb-dev/tbhbdev. Front, dispatch, and plugin plumbing pull requests under `<cli-root>` go to the operator before merge. The monorepo's general scaffolding, which the monorepo session owns and whose first pull request creates neither candidate CLI folder, isn't gated.
- **Undo:** a new operator decision.
- **Record:** the operator's decisions of 2026-11-25, relayed with D17, D18, and D20, and the operator's later decisions that day on the gate and on its carry-over to the monorepo.

### D18. The Go stack is cobra, pflag, and koanf, with a standard-library front

- **Decided:** 2026-11-25, by the operator.
- **Rule:** plugin binaries build their command trees with cobra, parse flags with pflag, and load config with koanf. Each command's constructor takes its dependencies explicitly, no package holds state or registers commands in `init()`, and koanf loads once in the input/output (I/O) shell and hands pure code a plain value. The front's dispatch stays pure and on the standard library, because it runs before any flag parsing and must pass unknown names and flags through untouched ([appendix K](K-go-stack.md)).
- **Undo:** a new operator decision. A later library swap touches each plugin's `io/app` and the config loader, and no pure package.
- **Record:** the operator's decisions of 2026-11-25, relayed with D17, D19, and D20.

### D17. The review tooling ports first, and implementation waits for the operator's read

- **Decided:** 2026-11-25, by the operator, reordering the first draft's waves.
- **Rule:** wave 1 ports the review tooling, the one unfinished part of tbhbdev: review runs, verdicts, posting reviews to GitHub, and the unbuilt pieces of the review-wire milestone plan. The launcher moves to wave 2, because the rest of tbhbdev works today. Protocol versioning (D14 and D16) and self-update (D15) stay as decided, and no implementation starts until the operator has read the merged design.
- **Undo:** a new operator decision.
- **Record:** the operator's decisions of 2026-11-25, relayed with D18 to D20.

### D16. The driver-host format is one versioned schema with a CI check

- **Decided:** 2026-11-25, by the operator.
- **Rule:** the run request and the run metadata are one schema file, `schemas/agent-protocol.json` in tools, with a version, and the driver and the host both validate against it. An additive change keeps the version. A breaking change bumps it, and the host keeps serving the previous version until the next bump, which waits until every listed machine serves the version it replaces. A CI job fails when the schema changes without a matching entry in `schemas/agent-protocol.decisions.toml`, as [appendix I](I-protocol.md#the-schema-version-check) specifies. The protocol version in D14 is this schema's version. A frozen legacy version 2 shape, today's exact fields, is what a writer sends a host too old to answer the protocol query, in the envelope and in the staged request file. The host writes the chosen version as each run's first file, and every reader validates that run's files against it.
- **Undo:** a new operator decision.
- **Record:** the operator's decision of 2026-11-25, relayed with D14 and D15.

### D15. tbhbdev updates itself at start when it's behind main

- **Decided:** 2026-11-25, by the operator.
- **Rule:** at start, a pure decision compares the build's commit with a cached main commit, kept at `$XDG_CACHE_HOME/tbhbdev/main.json` with a 10-minute time to live. When the build is behind, an I/O helper downloads the bundle beside the install, checks its SHA-256 hash and provenance, renames a whole versioned directory and then a link into place, and execs the new front. No failure blocks dispatch: a missing network, a bad hash, a read-only install, or a failed exec of the helper warns and runs the current build, a failed handoff puts the old link back, and in-band update work stops at a 3-second budget, after which a detached installer finishes it. `TBHBDEV_NO_SELF_UPDATE=1` turns it off. Event-driven sync stays the main path and writes the same cache. [Appendix J](J-self-update.md) has the details.
- **Undo:** a new operator decision. Setting `TBHBDEV_NO_SELF_UPDATE=1` holds one machine.
- **Record:** the operator's decision of 2026-11-25, relayed with D14 and D16.

### D14. Drivers and hosts check a protocol version, never a build

- **Decided:** 2026-11-25, by the operator, refining D6.
- **Rule:** a driver and a host agree on a protocol version, the version of the D16 schema, and never compare build hashes. A build mismatch is a logged warning and never a refusal, and no version mismatch blocks a launch. This replaces the exact build match in `same_version` and tools#400's retry of it, and the transition in [appendix I](I-protocol.md#the-transition) refuses nothing with a Python driver and a Go host, with a Go driver and a Python host, or with mixed bundle versions.
- **Before phase 1:** builds already installed can't be changed, so phase 1 ends with a rollout barrier: launches pause, every listed machine syncs, and launches resume when none runs an older build. A machine the sync can't reach is the one accepted exception to D6 until it's synced, and the operator signs the list of such machines in the barrier record.
- **Undo:** a new operator decision.
- **Record:** the operator's decision of 2026-11-25, relayed with D15 and D16.

### D13. Test time caps for Go

- **Decided:** 2026-11-25, by the tools maintainer role, under the operator's cap of 5 minutes with unit tests in seconds.
- **Rule:** gazelle maps `go_test` to `unit_go_test` in pure packages, which is `size = "small"` with `-test.timeout=10s`, and to `io_go_test` under `io/`, which is `size = "medium"`, so Bazel stops it at 300 seconds. A query test fails on any other Go test kind.
- **Undo:** change the macros. Each target picks up the new limits on its next run.
- **Record:** this design's pull request.

### D12. The agent wire protocol stays frozen during the port (superseded)

- **Decided:** 2026-11-25, by the tools maintainer role. Superseded the same day by D14 and D16.
- **Rule:** none now. The protocol changes under the schema's version rules in D16, and drivers and hosts choose a common version per launch under D14, instead of freezing until every host runs Go.
- **Undo:** not needed. D16's rules allow a freeze as a special case, since a release may make no breaking change.
- **Record:** this design's pull request.

### D11. Pure and I/O packages are told apart by directory

- **Decided:** 2026-11-25, by the tools maintainer role.
- **Rule:** I/O code lives in `cmd/`, under any `io/` directory, and in each plugin's `io/app` wiring package. Everything else under `internal/` is pure. No package takes the name `core` or `shell`.
- **Undo:** move packages and update the arch-lint, depguard, and ast-grep globs in one change.
- **Record:** this design's pull request.

### D10. Final names follow appendix G

- **Decided:** 2026-11-25, by the tools maintainer role, under D5.
- **Rule:** each ported command takes the final name in [appendix G](G-names-and-migration.md). `tbhbdev pr` gathers the pull request commands, `tbhbdev review` the review commands (D21), and `tbhbdev hook` the harness hooks.
- **Undo:** rename before callers move. After they move, a rename is a second migration.
- **Record:** this design's pull request.

### D9. The front hands off by exec

- **Decided:** 2026-11-25, by the tools maintainer role.
- **Rule:** the front calls `syscall.Exec`, so the plugin keeps the process ID, every descriptor, and signal delivery, and its exit status is the process's. Windows is out of scope, since the launcher depends on tmux and ssh.
- **Undo:** a child process with wrapped exit codes, as gh does, at the cost in [appendix C](C-alternatives.md).
- **Record:** this design's pull request.

### D8. The Go code lives in tbhb-dev/tools as one module (superseded)

- **Decided:** 2026-11-25, by the tools maintainer role. Superseded the same day by the operator's D22, which moves the Go code to tbhb-dev/tbhbdev.
- **Rule:** one `go.mod` at the tools root, module `go.tbhb.dev/tools`, with no `go.work`. One commit builds the bundle, the Go binaries, and the Python wheel it falls through to.
- **Undo:** not needed. D22 keeps one commit per bundle by pinning the tools commit in the monorepo.
- **Record:** this design's pull request.

### D7. Go-only pull requests don't run the Python suite

- **Decided:** 2026-11-25, by the operator.
- **Rule:** Go and Python targets sit in separate Bazel packages with disjoint inputs, with no catch-all target, so a pull request or merge queue run tests only what its change affects. Since D22 the Go code is in another repository, so a Go change never starts the tools suite, and inside the monorepo the tests that run the Python server take the pinned wheel as an input. The CI speed role owns CI wall time across repositories.
- **Undo:** one-way in practice. Merging the inputs again brings back the 300-second suite on every change.
- **Record:** the operator's decision of 2026-11-25.

### D6. No version mismatch ever blocks a launch

- **Decided:** 2026-11-25, by the operator.
- **Rule:** at every step of the transition, any driver can launch on any host. A host that can't serve a driver's request counts as a refusal too.
- **Undo:** a new operator decision.
- **Record:** the operator's decision of 2026-11-25.

### D5. The Go port has no `alpha`

- **Decided:** 2026-11-25, by the operator.
- **Rule:** Go commands carry final names, such as `tbhbdev agent exec` and `tbhbdev pr create`. `tbhbdev alpha <command>` keeps working only through the Python fallthrough until its command ports.
- **Undo:** a new operator decision.
- **Record:** the operator's decision of 2026-11-25.

### D4. tbhbdev moves to Go as plugin binaries

- **Decided:** 2026-11-25, by the operator.
- **Rule:** tbhbdev becomes a modular monorepo of plugin binaries in Go, modeled on gh after a study of its pitfalls. The Python pure-core refactor stops. The first build is the front and its plugin plumbing, with unported names falling through to the Python tbhbdev. Pure packages and thin I/O packages stay apart, and one commit builds one release bundle.
- **Undo:** a new operator decision. Ported commands would need their Python code back.
- **Record:** the operator's decision of 2026-11-25. No record in the decision folder covered a Go port at the time of the outline's read.
