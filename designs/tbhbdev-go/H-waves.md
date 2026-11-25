---
type: Design Appendix
title: tbhbdev Go port waves
description: What the tbhbdev Go port moves in each wave, with line counts, the review tooling that wave 1 carries, and how the work splits into parallel pull requests under the size gate.
tags: [tbhbdev, go, plan, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words agy userconfig gitleaks runfiles asyncio depguard cobra koanf -->

# tbhbdev Go port waves

Back to [the design](README.md).

Line counts come from the design outline, read at tools `75cbfd83` on 2026-11-25, except the wave 1 review modules, read at tools `bf420ff` the same day. They're `wc -l` of whole files, comments and blank lines included. The Python source is 37,745 lines under `src/tbhbdev`, and the tests are 95,572 lines in 150 files, of which `alpha` alone is 29,364 lines.

No wave starts until the operator has read the merged design (D17).

## Waves

| Wave | What ports | Python lines | Why here |
| --- | --- | --- | --- |
| 0 | The front, the plugin runner on cobra, the koanf loader, the schema package, the contract library, the checks, the bundle, the installer, and self-update, all new | none | Everything else stands on it |
| 1 | `alpha review`: `review` (309), `review_doc` (427), `review_policy` (242), and `review_prompt` (121), with `review_data/` and `review_prompt_data/` | 1,099 | The review tooling is the unfinished part of tbhbdev (D17) |
| 1 | The review run's post: `agent/review_post.py` (177), and `post_review` in `run_host.py` once tools#399 merges | 177, plus part of `run_host` | A review run posts its own verdict |
| 1 | What the review code imports: the content check in `content_vet` (186) with the parts of `content` and `content_model` it calls, the app and review tokens in `gh_app` (815), the parts of `pull_request` that `review_post.py` calls, and the GitHub client | unverified, part of each module | Ported as Go library packages beside the Python libraries, both tested on the same fixtures |
| 2 | `alpha agent`, the launcher | 9,769, of which `decisions/` is 2,912 | The launcher works today, and `decisions/` is already pure, so it needs only translation |
| 2 | The libraries the launcher imports: `version` (83), `userconfig` (203), `telemetry` (147), the `secrets` library (701), `gitleaks_check` (84), and constants from `identity`, `repo`, and `host` | about 1,250 | The launcher can't run without them |
| 2 | The launcher's uses of other `alpha` modules: `brief` (`driver.py`), `gh_app` (`credentials.py`), `install` (`worktree.py`), and `pull_request` (`open_pr.py`) | unverified, part of each module | Ported as library packages with no command yet, so Go never calls `alpha` (D5). The review pieces are in Go from wave 1 |
| 3 | `identity`, the `guard_*` modules, `gh_shim`, `state_guard`, and `session_context` | 3,137 and 4,283 | Hooks run on every tool call, so their start time matters most |
| 3 | `pull_request` (2,199), `squash_merge` (798), the rest of `gh_app` (815), `close` (453), and the `repo_*` modules | about 4,300, plus `repo_*` | Large GitHub adapters, sharing the wave 1 client |
| 3 | `loop` | 2,255 | |
| 4 | `install` | 1,521 | It becomes the bundle installer's full form |
| 4 | `brief`, `verify-range`, `version-check`, `pre-push-guard`, and `pr-stats` | about 1,100 | |
| 4 | `admin` (1,484), the `secret` command (160), `agy` (276), `adr` with `adr_decision` (250), and `issue` with `issue_import` (282) | about 2,450 | |
| none | `cue` (156) | | Go runs no `cue` binary (D20), so the command that installs it stays with the Python content libraries and goes when they go |
| none | `analytics` (727) and `host` (2,395) | | D2 |
| none | `gitleaks_check`, `prose_check`, `content`, `content_model`, and `content_vet` as Python libraries | | Other repositories import them (unverified which), so they stay until those callers move |

`policy.py` in `decisions/` imports `secrets` and `host_tools`, which sit outside `decisions/`. Its purity gets a check before it ports as a pure package. The wave 2 Go copy of `gitleaks_check` and the wave 1 Go content check live beside their Python libraries until the libraries' callers move.

The Python pure-core refactor stops (D4). The outline read tools#385 as its only open pull request, and it has since merged as the review document types and pure functions (per the tools inventory), so wave 1 ports it with `review_doc`.

## Wave 1: the review tooling

The lists below come from the tools inventory of 2026-11-25, which read the git log of tools main and the `src/tbhbdev` tree. Merged means on tools main with the pull request's number in a commit subject.

### In Python today

- **Review runs**, in `alpha/agent/` (`launcher`, `driver`, `run_host`, `render`, and `admission`): read-only tokens with no approval routes (tools#238), `--ref` required for a review `exec` (tools#249), launch slots kept for reviews (tools#321), no second live review of one commit (tools#340), both runs launched for a declared review pair (tools#347), sessions that launch their own reviews (tools#370), and reviewer-role runs refused from unlisted sessions (tools#265).
- **The verdict line.** A review run ends with `VERDICT: GO|NO-GO <sha>` (tools#395), and the run host posts it by the older path from tools#278.
- **Review documents:** types and pure functions (tools#385) and parse and render (tools#388) in `review_doc.py`, the schema `schemas/review.cue` (tools#384), and the content check before posting in `content_vet.py` (tools#310).
- **Review commands:** `draft` and `check` (tools#33), `post`, `dismiss`, and `reply` (tools#392) in `review.py`, the policy check in `review_policy.py` (tools#277), the floor template and schemas (tools#268), the org floor checklist (tools#274), the prompt templates in `review_prompt.py` and `review_prompt_data/` (tools#393), and the review token and commit fetch (tools#212 and tools#215).
- **In flight:** tools#399 posts review run verdicts from the run host, through `review_post.py`, `post_review` in `run_host.py`, and `review check N --no-lint`. At the inventory's read it was open, approved, and queued, and it lands unchanged.

After tools#399, Python gets no new review features, so wave 1 ports the review code as it stands then.

### Unbuilt

The unbuilt list comes from the milestone plan and is not confirmed against a tracker.

- **The M4 proof.** One real review run posts its verdict as a GitHub review at the head it judged, after tools#399 merges. If it first passes on the Python path, wave 1 repeats it on the Go path, and wave 1 is done only when the Go path passes.
- **The M5 merge-gate switch.** A design for the operator on which identity approves, the rulesets, and retiring `tbhb-approve`. It's a decision, not code, and any code it calls for lands in Go.
- **The paired review launch and its summary.** tools#358 and tools#365 closed, and no matching subject is on main, so they read as unmerged (checked only against the git log). Wave 1 builds them as a new `tbhbdev review` subcommand.
- **Debt from the tools#399 and tools#400 reviews:** a guard against an empty brief, proxy variables that `version_heal` drops, and a lint for the review prompts. Each lands as a fix and a test in the Go package that ports its module.

### Review runs before the launcher ports

- **Launching.** The Go review code imports nothing from the launcher. To start a review run, it calls `tbhbdev agent exec` by its final name through `TBHBDEV_PATH`.
- **Python answers `agent`.** From wave 1, the Python package answers the top-level name `agent` with the `alpha agent` code, so Go names no `alpha` command (D5). In wave 2 the Go `agent` plugin takes the name, and the review code doesn't change.
- **Admission.** Rules that read the launcher's run state, such as reserved slots and one live review per commit, stay in the Python launcher until wave 2. Go doesn't copy them.
- **Posting.** Once `tbhbdev review post` ships, the Python run host's `post_review` hands the verdict to it by its final name, so one poster serves both paths.

## Slices

The size gate allows 400 added code lines per pull request, tests included. With Python tests at about 2.5 times the source, one slice is about 120 to 150 Python source lines with their tests, and fewer once Go's length is measured.

### Go scaffolding

About five pull requests, one after another:

1. The front with `internal/dispatch` and `internal/plugin`.
2. The stamp and the bundle test.
3. `internal/plugin/io/run` with the cobra root, and `internal/config/io/load` with koanf ([appendix K](K-go-stack.md)).
4. The checks with their planted fixtures, the stack rules included.
5. `internal/schemas`, with the Configure Unify Execute (CUE) files moved over and in-process validation.

Each scaffolding pull request goes to the operator before it merges (D19). The phase 1 Python release, which adds the protocol schema and the `protocol-version` job, and the rules ast-grep macro run beside them.

### Wave 1 in parallel

The review modules make about 15 slices, in three layers by their imports (an estimate from the line counts):

1. `review_doc` types, parse, and render, the prompt templates, and the GitHub client. About 6 slices at once.
2. `review_policy`, the content check on `internal/schemas`, and the review and app tokens.
3. The `review` commands, the post, and the paired launch and summary.

### Wave 2

Serial first: `internal/agent/model` from `model.py` (280 lines), then `internal/agent/protocol` on the phase 1 schema. Then the 12 `decisions/` modules make about 20 slices, in three layers by their imports:

1. `repo` (76), `runfiles_names` (42), `launcher` (36), `limits` (81), `run_state` (103), and `auth` (121), which need only `model`. About 8 slices at once.
2. `config` (603), `request` (323), `remote` (531), and `policy` (482).
3. `argv` (337) and `driver` (176).

The adapters follow the pure packages they call, in about 30 more slices with about 6 at once: `transport` (189), `process_group` (344), `worktree` (644), `push` (549), `credentials` (350), `run_host` (1,463), `driver` (754), and `ssh_serve` (189). The envelope corpus lands with the first `ssh_serve` slice.

Host load rules cap real concurrency, not the code (unverified as current).

### Why parallel slices don't collide

- Each slice adds new package directories, and gazelle writes each package's own `BUILD.bazel`, so two slices never edit one build file.
- The arch-lint, depguard, and ast-grep configs match by glob, so a new package needs no config edit.
- A subcommand slice adds one `AddCommand` line to its plugin's root constructor, the only line two slices share, and a rebase settles it.
- Only the serial steps and dependency bumps touch `go.mod`, `MODULE.bazel`, and the lint configs, one at a time.
- Each slice ports the Python tests for its module as Go table tests in the same pull request, so the Python suite stays the behavior spec until the module's Python code goes.

### Order inside a module

1. Pure types and functions, with their table tests.
2. The adapter, with its `io_go_test` against real tools in a temporary root.
3. The `io/app` wiring and the command, behind its final name.
4. The corpus or golden comparison against the Python command from the same commit.
