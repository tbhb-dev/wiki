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

<!-- cspell:words gazelle depguard ITSM syscall unported libexec cobra pflag koanf cuelang -->

# tbhbdev Go port decisions

Back to [the design](README.md).

## Open

### D1. Do releases ship as a GitHub Release bundle, checked by a GitHub artifact attestation and a SHA-256 manifest, with no signing key held by tbhb-dev?

- **Recommendation:** yes.
- **Why:** this replaces `uv tool install git+https://github.com/tbhb-dev/tools` with a prebuilt bundle, which changes what a host trusts at install time. Self-update (D15) installs a new bundle on its own at start, so the check must need nothing an unattended run lacks. An attestation ties each archive to the tools release workflow and commit, and needs no key to keep or rotate. Neither afakt nor awrel signs or attests today (unverified beyond the outline's grep).
- **If yes:** plan step 4 adds `actions/attest-build-provenance` to the release job, which publishes every main commit's bundle and the `main.json` pointer. The installer, sync, and self-update all run `gh attestation verify` and check the archive and manifest hashes before they swap the link. A failed check warns and keeps the current build, as D15 requires.
- **If no:** the operator names a held key. The admin VM role creates it in 1Password, the release job signs with it, and every bundle carries the public key that its self-update checks the next bundle with, which adds a rotation runbook.
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

### D21. The review commands take the top-level name `review`

- **Decided:** 2026-11-25, by the tools maintainer role, under D17 and refining D10.
- **Rule:** `alpha review draft`, `check`, `policy`, `post`, `dismiss`, and `reply` port as `tbhbdev review` in their own plugin, `tbhbdev-review`, and not as `pr review` in `tbhbdev-pr`. Wave 1 ships the review commands alone. A `tbhbdev-pr` that answered only `review` would send every other `pr` word to a Go usage error until wave 3.
- **Undo:** rename before callers move. After they move, a rename is a second migration.
- **Record:** this design's pull request.

### D20. Go validates schemas in-process with `cuelang.org/go`

- **Decided:** 2026-11-25, by the operator.
- **Rule:** the Go port never runs the `cue` binary. Every schema check goes through `cuelang.org/go` in a pure package, with the schema embedded by `go:embed`, including the issue and pull request templates and the D16 run request and run metadata schema. depguard and ast-grep rules forbid running `cue` and forbid `os/exec` in pure packages ([appendix K](K-go-stack.md#the-rules-that-hold-it)), and the Python `src/tbhbdev/schemas/*.cue` files are the source that moves over.
- **Undo:** a new operator decision.
- **Record:** the operator's decisions of 2026-11-25 on the wave order and the stack, relayed with D17 to D19.

### D19. Scaffolding pull requests go to the operator before merge

- **Decided:** 2026-11-25, by the operator.
- **Rule:** each Go scaffolding pull request goes to the operator before it merges. These are the serial pull requests in [appendix H](H-waves.md#go-scaffolding) that set up the front, the plugin runner, the stack, the schema package, and the checks.
- **Undo:** a new operator decision.
- **Record:** the operator's decisions of 2026-11-25, relayed with D17, D18, and D20.

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
- **Rule:** the run request and the run metadata are one schema file, `schemas/agent-protocol.json` in tools, with a version, and the driver and the host both validate against it. An additive change keeps the version. A breaking change bumps it, and the host keeps serving the previous version until the next bump, which waits until every listed machine serves the version it replaces. A CI job fails when the schema changes without a matching entry in `schemas/agent-protocol.decisions.toml`, as [appendix I](I-protocol.md#the-schema-version-check) specifies. The protocol version in D14 is this schema's version.
- **Undo:** a new operator decision.
- **Record:** the operator's decision of 2026-11-25, relayed with D14 and D15.

### D15. tbhbdev updates itself at start when it's behind main

- **Decided:** 2026-11-25, by the operator.
- **Rule:** at start, a pure decision compares the build's commit with a cached main commit, kept at `$XDG_CACHE_HOME/tbhbdev/main.json` with a 10-minute time to live. When the build is behind, an I/O helper downloads the bundle beside the install, checks its SHA-256 hash and provenance, renames a whole versioned directory and then a link into place, and execs the new front. No failure refuses a run: a missing network, a bad hash, or a read-only install warns and runs the current build. `TBHBDEV_NO_SELF_UPDATE=1` turns it off. Event-driven sync stays the main path and writes the same cache. [Appendix J](J-self-update.md) has the details.
- **Undo:** a new operator decision. Setting `TBHBDEV_NO_SELF_UPDATE=1` holds one machine.
- **Record:** the operator's decision of 2026-11-25, relayed with D14 and D16.

### D14. Drivers and hosts check a protocol version, never a build

- **Decided:** 2026-11-25, by the operator, refining D6.
- **Rule:** a driver and a host agree on a protocol version, the version of the D16 schema, and never compare build hashes. A build mismatch is a logged warning and never a refusal, and no version mismatch blocks a launch. This replaces the exact build match in `same_version`, and the transition in [appendix I](I-protocol.md#the-transition) refuses nothing with a Python driver and a Go host, with a Go driver and a Python host, or with mixed bundle versions.
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

### D8. The Go code lives in tbhb-dev/tools as one module

- **Decided:** 2026-11-25, by the tools maintainer role.
- **Rule:** one `go.mod` at the tools root, module `go.tbhb.dev/tools`, with no `go.work`. One commit builds the bundle, the Go binaries, and the Python wheel it falls through to.
- **Undo:** move the module to a new repository, which splits the bundle across two commits.
- **Record:** this design's pull request.

### D7. Go-only pull requests don't run the Python suite

- **Decided:** 2026-11-25, by the operator.
- **Rule:** Go and Python targets sit in separate Bazel packages with disjoint inputs, with no catch-all target, so a pull request or merge queue run tests only what its change affects. The CI speed role owns CI wall time across repositories.
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
