---
type: Design Appendix
title: tbhbdev Go port waves
description: What the tbhbdev Go port moves in each wave, with line counts, and how the work splits into parallel pull requests under the size gate.
tags: [tbhbdev, go, plan, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words agy userconfig gitleaks runfiles asyncio depguard -->

# tbhbdev Go port waves

Back to [the design](README.md).

Line counts come from the design outline, read at tools `75cbfd83` on 2026-11-25. They're `wc -l` of whole files, comments and blank lines included. The Python source is 37,745 lines under `src/tbhbdev`, and the tests are 95,572 lines in 150 files, of which `alpha` alone is 29,364 lines.

## Waves

| Wave | What ports | Python lines | Why here |
| --- | --- | --- | --- |
| 0 | The front, the contract library, the checks, the bundle, the installer, and self-update, all new | none | Everything else stands on it |
| 1 | `alpha agent`, the launcher | 9,769, of which `decisions/` is 2,912 | `decisions/` is already pure and needs only translation |
| 1 | The libraries the launcher imports: `version` (83), `userconfig` (203), `telemetry` (147), the `secrets` library (701), `gitleaks_check` (84), and constants from `identity`, `repo`, and `host` | about 1,250 | The launcher can't run without them |
| 1 | The launcher's uses of other `alpha` modules: `brief` (`driver.py`), `gh_app` (`credentials.py`), `install` (`worktree.py`), `pull_request` (`open_pr.py`, `review_post.py`), and `review` (`review_post.py`) | unverified, part of each module | Ported as library packages with no command yet, so Go never calls `alpha` (D5) |
| 2 | `identity`, the `guard_*` modules, `gh_shim`, `state_guard`, and `session_context` | 3,137 and 4,283 | Hooks run on every tool call, so their start time matters most |
| 2 | `pull_request` (2,199), `squash_merge` (798), `gh_app` (815), `close` (453), `review` (309), `review_doc` (427), and the `repo_*` modules | about 5,000, plus `repo_*` | Large GitHub adapters, sharing one client |
| 2 | `loop` | 2,255 | |
| 3 | `install` | 1,521 | It becomes the bundle installer's full form |
| 3 | `brief`, `verify-range`, `version-check`, `pre-push-guard`, and `pr-stats` | about 1,100 | |
| 3 | `admin` (1,484), the `secret` command (160), `agy` (276), `adr` with `adr_decision` (250), `issue` with `issue_import` (282), and `cue` (156) | about 2,400 | |
| none | `analytics` (727) and `host` (2,395) | | D2 |
| none | `gitleaks_check`, `prose_check`, `content`, `content_model`, and `content_vet` as Python libraries | | Other repositories import them (unverified which), so they stay until those callers move |

`policy.py` in `decisions/` imports `secrets` and `host_tools`, which sit outside `decisions/`. Its purity gets a check before it ports as a pure package. The wave 1 Go copy of `gitleaks_check` lives beside the Python library until the library's callers move, and both test against the same fixtures.

The Python pure-core refactor stops (D4). Its only verified open pull request, tools#385 (commit `da154050`, 121 source and 104 test lines), ports almost line for line if it lands, and the port doesn't wait for it.

## Slices

The size gate allows 400 added code lines per pull request, tests included. With Python tests at about 2.5 times the source, one slice is about 120 to 150 Python source lines with their tests, and fewer once Go's length is measured.

### Serial first

About four pull requests, one after another: the front with `internal/dispatch` and `internal/plugin`, then the stamp and the bundle test, then the checks with their planted fixtures, then `internal/agent/model` from `model.py` (280 lines) with `internal/agent/protocol` on the phase 1 schema. The phase 1 Python release, which adds the schema and the `protocol-version` job, and the rules ast-grep macro run beside them.

### Then in parallel

The 12 `decisions/` modules make about 20 slices, in three layers by their imports:

1. `repo` (76), `runfiles_names` (42), `launcher` (36), `limits` (81), `run_state` (103), and `auth` (121), which need only `model`. About 8 slices at once.
2. `config` (603), `request` (323), `remote` (531), and `policy` (482).
3. `argv` (337) and `driver` (176).

The adapters follow the pure packages they call, in about 30 more slices with about 6 at once: `transport` (189), `process_group` (344), `worktree` (644), `push` (549), `credentials` (350), `run_host` (1,463), `driver` (754), and `ssh_serve` (189). The envelope corpus lands with the first `ssh_serve` slice.

Host load rules cap real concurrency, not the code (unverified as current).

### Why parallel slices don't collide

- Each slice adds new package directories, and gazelle writes each package's own `BUILD.bazel`, so two slices never edit one build file.
- The arch-lint, depguard, and ast-grep configs match by glob, so a new package needs no config edit.
- Only the serial steps and dependency bumps touch `go.mod`, `MODULE.bazel`, and the lint configs, one at a time.
- Each slice ports the Python tests for its module as Go table tests in the same pull request, so the Python suite stays the behavior spec until the module's Python code goes.

### Order inside a module

1. Pure types and functions, with their table tests.
2. The adapter, with its `io_go_test` against real tools in a temporary root.
3. The `io/app` wiring and the command, behind its final name.
4. The corpus or golden comparison against the Python command from the same commit.
