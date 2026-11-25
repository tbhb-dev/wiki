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

<!-- cspell:words gazelle depguard ITSM syscall unported -->

# tbhbdev Go port decisions

Back to [the design](README.md).

## Open

### D1. Do releases ship as a GitHub Release bundle, checked by a GitHub artifact attestation and a SHA-256 manifest, with no signing key held by tbhb-dev?

- **Recommendation:** yes.
- **Why:** this replaces `uv tool install git+https://github.com/tbhb-dev/tools` with a prebuilt bundle, which changes what a host trusts at install time. An attestation ties each archive to the tools release workflow and commit, and needs no key to keep or rotate. Neither afakt nor awrel signs or attests today (unverified beyond the outline's grep).
- **If yes:** plan step 4 adds `actions/attest-build-provenance` to the release job, and the installer runs `gh attestation verify` and checks the manifest hashes before it swaps the link.
- **If no:** the operator names a held key. The admin VM role creates it in 1Password, the release job signs with it, and every installer carries the public key, which adds a rotation runbook.
- **Reversible:** yes. A later release can add a held signature beside the attestation, since the installer checks whatever the manifest declares.

### D2. Does `tbhbdev host` retire without a port, and does `tbhbdev analytics` leave the bundle as its own Python tool, so the Python fallthrough can end?

- **Recommendation:** yes.
- **Why:** `host` is 2,395 lines that a memory note calls frozen until ops replaces it (unverified as current). `analytics` is 727 lines of PostgreSQL migrations with no tie to the launcher. Porting either costs more than it returns, and keeping either in the bundle keeps a Python interpreter in every install for good.
- **If yes:** `host` stays in the fallthrough until ops ships its replacement, then goes. `analytics` moves to its own package with its own console script before plan step 8. The library-only modules stay in Python until their callers in other repositories move.
- **If no:** `host` or `analytics` joins wave 3 as Go, about 2,400 or 700 more Python lines to port, or the bundle keeps the Python wheel with no end date.
- **Reversible:** yes until the Python code is deleted, then only by restoring it from history.

### D3. May the front run plugins from outside its release bundle, but only from absolute paths listed in an operator-owned file, and never from a `PATH` search?

- **Recommendation:** yes.
- **Why:** graph and the private ITSM repository add tools today through Python entry points in the `tbhbdev.plugins` group, and they need a way in once the fallthrough ends. Agent hooks run `tbhbdev` on every tool call, so what it may run is a trust boundary. gh dropped its `PATH` search six days after it shipped ([appendix B](B-gh-pitfalls.md#11-path-discovery-dropped-after-six-days)).
- **If yes:** the front reads `$XDG_CONFIG_HOME/tbhbdev/plugins.toml` only when no bundle plugin matches. Each entry is a name and an absolute path, and a name that a built-in or bundle plugin uses is reported, never run.
- **If no:** only the bundle's plugins run. graph and the ITSM repository ship their own commands under their own names once the fallthrough ends.
- **Reversible:** yes. Removing the file read narrows the boundary again, and the entries just stop working.

## Decided

### D13. Test time caps for Go

- **Decided:** 2026-11-25, by the tools maintainer role, under the operator's cap of 5 minutes with unit tests in seconds.
- **Rule:** gazelle maps `go_test` to `unit_go_test` in pure packages, which is `size = "small"` with `-test.timeout=10s`, and to `io_go_test` under `io/`, which is `size = "medium"`, so Bazel stops it at 300 seconds. A query test fails on any other Go test kind.
- **Undo:** change the macros. Each target picks up the new limits on its next run.
- **Record:** this design's pull request.

### D12. The agent wire protocol stays frozen during the port

- **Decided:** 2026-11-25, by the tools maintainer role.
- **Rule:** drivers send envelope `v` 1 with request `v` 2 until every host runs a bundle whose forced command points at the Go server. Later changes reach every host before any driver sends them.
- **Undo:** two-way. A protocol change waits for the fleet, it doesn't break it.
- **Record:** this design's pull request.

### D11. Pure and I/O packages are told apart by directory

- **Decided:** 2026-11-25, by the tools maintainer role.
- **Rule:** I/O code lives in `cmd/`, under any `io/` directory, and in each plugin's `io/app` wiring package. Everything else under `internal/` is pure. No package takes the name `core` or `shell`.
- **Undo:** move packages and update the arch-lint, depguard, and ast-grep globs in one change.
- **Record:** this design's pull request.

### D10. Final names follow appendix G

- **Decided:** 2026-11-25, by the tools maintainer role, under D5.
- **Rule:** each ported command takes the final name in [appendix G](G-names-and-migration.md). `tbhbdev pr` gathers the pull request commands and `tbhbdev hook` gathers the harness hooks.
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
