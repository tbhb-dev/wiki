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

<!-- cspell:words agy userconfig gitleaks runfiles asyncio depguard cobra koanf ghapi unresolve -->

# tbhbdev Go port waves

Back to [the design](README.md).

Line counts come from the design outline, read at tools `75cbfd83` on 2026-11-25, except the wave 1 review modules, counted at tools `c02551d` on 2026-11-26. They are `wc -l` of whole files, comments and blank lines included. The Python source is 37,745 lines under `src/tbhbdev`, and the tests are 95,572 lines in 150 files, of which `alpha` alone is 29,364 lines.

No wave starts until tbhb has read the merged design (D17). After the Python import, each wave's changes to Go and Python land in tbhb-dev/tbhbdev under `factory/cli` (D22).

## Waves

| Wave | What ports | Python lines | Why here |
| --- | --- | --- | --- |
| 0 | The front, the cobra wiring, the koanf loader, the schema package, the GitHub client package, the checks, the bundle, the installer, and self-update, all new | none | Everything else stands on it |
| 1 | `alpha review`: `review` (359), `review_doc` (333), `review_policy` (242), and `review_prompt` (121), with `review_data/` and `review_prompt_data/` | 1,055 | The review tooling is the unfinished part of tbhbdev (D17) |
| 1 | The review run's post: `agent/review_post.py` and the run host's `post_review` call to it | 257, plus part of `run_host` | A review run posts its own verdict |
| 1 | What the review code imports: the content check in `content_vet` (186) with the parts of `content` and `content_model` it calls, the app and review tokens in `gh_app` (815), the parts of `pull_request` that `review_post.py` calls, and the GitHub client on go-github (D24) | Selected functions | Ported as Go library packages beside the Python libraries, both tested on the same fixtures |
| 2 | `alpha agent`, the launcher, with `agent env` and `agent gh-shim` passed through to `tbhbdev-py` until wave 3 (D26) | 9,769, of which `decisions/` is 2,912 | The launcher works today, and `decisions/` is already pure, so it needs only translation |
| 2 | The libraries the launcher imports: `version` (83), `userconfig` (203), `telemetry` (147), the `secrets` library (701), `gitleaks_check` (84), and constants from `identity`, `repo`, and `host` | about 1,250 | The launcher cannot run without them |
| 2 | The launcher's uses of other `alpha` modules: `brief` (`driver.py`), `gh_app` (`credentials.py`), `install` (`worktree.py`), and `pull_request` (`open_pr.py`) | Selected functions | Ported as library packages with no command yet, so Go never calls `alpha` (D5). The review pieces are in Go from wave 1 |
| 3 | `identity`, the `guard_*` modules, `gh_shim`, `state_guard`, and `session_context` | 3,137 and 4,283 | Hooks run on every tool call, so their start time matters most |
| 3 | `pull_request` (2,199), `squash_merge` (798), the rest of `gh_app` (815), `close` (453), and the `repo_*` modules | about 4,300, plus `repo_*` | Large GitHub adapters, sharing the wave 1 client |
| 3 | `loop` | 2,255 | |
| 4 | `install` | 1,521 | It becomes the bundle installer's full form |
| 4 | `brief`, `verify-range`, `version-check`, `pre-push-guard`, and `pr-stats` | about 1,100 | |
| 4 | `admin` (1,484), the `secret` command (160), `agy` (276), `adr` with `adr_decision` (250), and `issue` with `issue_import` (282) | about 2,450 | |
| none | `cue` (156) | | Go runs no `cue` binary (D20), so the command that installs it stays with the Python content libraries and goes when they go |
| later | `analytics` | 727 | Stays in the bundle and runs through the Python fallthrough until it ports (D2) |
| none | `host` | 2,395 | Retired on 2026-11-08 after its work moved to ops' Ansible. Removed from the fallback command list (D2) |
| none | `gitleaks_check`, `prose_check`, `content`, `content_model`, and `content_vet` as Python libraries | | Keep the Python libraries until their external callers move |

`policy.py` in `decisions/` imports `secrets` and `host_tools`, which sit outside `decisions/`. Its purity gets a check before it ports as a pure package. The wave 2 Go copy of `gitleaks_check` and the wave 1 Go content check live beside their Python libraries until the libraries' callers move.

The Python pure-core refactor stops (D4). Tools#385 merged the review document types and pure functions as `da154050`, so wave 1 ports them with `review_doc`.

## Wave 1: the review tooling

This inventory was checked on 2026-11-26 against a fresh fetch of [tools main at `c02551d`](https://github.com/tbhb-dev/tools/commit/c02551d5089ecbc08404f565b73cec4724a60dea) and GitHub pull request records.

### In Python today

- **Review runs**, in `alpha/agent/` (`launcher`, `driver`, `run_host`, `render`, and `admission`): read-only tokens with no approval routes (tools#238), `--ref` required for a review `exec` (tools#249), launch slots kept for reviews (tools#321), no second live review of one commit (tools#340), both runs launched for a declared review pair (tools#347), sessions that launch their own reviews (tools#370), and runs started with a reviewer `--role` refused from unlisted sessions (tools#265), where `--role` is the existing `agent exec` flag.
- **The verdict line.** The current Python prompt asks for `VERDICT: GO <sha>` or `VERDICT: BLOCK <sha>`. The run host also accepts legacy `NO-GO`. The prompt fix in wave 1 brings it in line with the Go contract below. [Prompt](https://github.com/tbhb-dev/tools/blob/c02551d5089ecbc08404f565b73cec4724a60dea/src/tbhbdev/alpha/review_prompt_data/common.txt)
- **The run host's poster.** tools#278 added `agent/review_post.py`, and tools#399 wired it to the run host and initially posted comment reviews. tools#430 switched the host to approvals or change requests. At `c02551d` the host checks the draft's commit against the run's pinned commit, checks the current head before approving, and records the post in a `review` run file so a retry skips it. tools#459 added host attribution, tools#460 removed the verdict line from the posted body, and tools#462 unified body validation with the manual command. [Poster source](https://github.com/tbhb-dev/tools/blob/c02551d5089ecbc08404f565b73cec4724a60dea/src/tbhbdev/alpha/agent/review_post.py)
- **Review documents:** types and pure functions (tools#385) and parse and render (tools#388) in `review_doc.py`, the schema `schemas/review.cue` (tools#384), and the content check before posting in `content_vet.py` (tools#310).
- **Review commands:** `draft` and `check` (tools#33), `post`, `dismiss`, and `reply` (tools#392) in `review.py`, the policy check in `review_policy.py` (tools#277), the floor template and schemas (tools#268), the org floor checklist (tools#274), and the prompt templates in `review_prompt.py` and `review_prompt_data/` (tools#393). tools#431 also added `threads`, `resolve-thread`, and `unresolve-thread`. [Command source](https://github.com/tbhb-dev/tools/blob/c02551d5089ecbc08404f565b73cec4724a60dea/src/tbhbdev/alpha/review.py)
- **The review-run token.** tools#278 added `credentials.review_post_token`. It gives the run host a posting token scoped to one repository. [Credential source](https://github.com/tbhb-dev/tools/blob/c02551d5089ecbc08404f565b73cec4724a60dea/src/tbhbdev/alpha/agent/credentials.py)

Wave 1 re-reads tools main on the day it starts and follows the Go review contract below. After that day Python gets no new review features.

### Milestone work

- **The M4 proof.** Wave 1 ends when a real Go review run posts its GO or HOLD verdict as a comment review on the exact head it judged.
- **The M5 merge-gate switch.** tools#441 merged the review-gate evaluator. The proposed `review verify` command in tools#458 closed without merging. Its pull request describes the required-status trial before retirement of `tbhb-approve`. Wave 1 keeps the Go review contract below. [Evaluator](https://github.com/tbhb-dev/tools/pull/441) and [verifier](https://github.com/tbhb-dev/tools/pull/458)
- **The paired review launch and its summary.** tools#358 and tools#365 both closed without merging, and the command is absent from tools main at `c02551d`. Wave 1 builds it as a new `tbhbdev review` subcommand. [tools#358](https://github.com/tbhb-dev/tools/pull/358) and [tools#365](https://github.com/tbhb-dev/tools/pull/365)
- **Review debt.** The driver rejects an empty brief when it restarts, and tools#437 added a missing-ref check. `version_heal.read_origin` passes a fixed environment to Git, which excludes proxy variables. The wave 1 slices carry the prompt lint and the relevant brief and environment cases into their tests. [Driver](https://github.com/tbhb-dev/tools/blob/c02551d5089ecbc08404f565b73cec4724a60dea/src/tbhbdev/alpha/agent/driver.py) and [version recovery](https://github.com/tbhb-dev/tools/blob/c02551d5089ecbc08404f565b73cec4724a60dea/src/tbhbdev/alpha/agent/version_heal.py)

### Review runs before the launcher ports

- **Launching.** The Go review code imports nothing from the launcher. To start a review run, it calls `tbhbdev agent exec` by its final name through `TBHBDEV_PATH`.
- **Python answers `agent`.** From the phase 1 release, before wave 1, the Python package answers the top-level name `agent` with the `alpha agent` code, so Go names no `alpha` command (D5). In wave 2 the Go `agent` group takes the name, and the review code does not change.
- **Admission.** Rules that read the launcher's run state, such as reserved slots and one live review per commit, stay in the Python launcher until wave 2. Go does not copy them.
- **Posting.** Once `tbhbdev review post` ships, the Python run host hands the verdict to it by its final name, under the contract below, so one poster serves both paths.

### The review contract wave 1 keeps

Wave 1 moves code, not review authority. ADR 0004 and ADR 0009 stay the rules, and each item below is a test in the wave 1 slices.

- **Verdict words.** A Go verdict is `GO` or `HOLD`, as ADR 0004 names them, and the Go prompt asks for `VERDICT: GO|HOLD <sha>`. The Go parser accepts both vocabularies: Go's `GO` and `HOLD`, and Python's `GO`, `BLOCK`, and legacy `NO-GO`. It maps `BLOCK` and `NO-GO` to `HOLD` so negative verdicts from Python runs in flight when Go takes over still post. It logs a warning for `NO-GO`. Wave 1 changes the imported Python prompt to ask for `HOLD`. Table test: `GO`, `HOLD`, `BLOCK`, `NO-GO`, a lowercase word, a missing commit, and a short commit. [Python verdict parser](https://github.com/tbhb-dev/tools/blob/c02551d5089ecbc08404f565b73cec4724a60dea/src/tbhbdev/alpha/agent/review_post.py#L26)
- **The exact judged head.** The verdict line's commit is the head the run judged, the run's `--ref`. The poster posts on that commit and never on the pull request's current head, and if the two differ it still posts on the judged one, which marks the verdict stale for the head that moved. That differs from the Python poster's check that an approval still targets the current head, so a verdict on a superseded head still reaches the pull request as ADR 0004's record of that exact head. Test: a head that moves during the run.
- **The submitted commit matches the run.** Before it posts, the run-host path compares the draft's submitted commit, with the commit in the run's request as the host wrote it, the trusted run metadata, and refuses to post on any difference, as `review_post.validate` does today. It never takes the commit from the review run's own output. Test: a fixture draft whose submitted commit differs from the run's request posts nothing, logs the refusal, and leaves the run's exit status intact. That is a different case from the head that moves, where the draft and the run agree and the pull request moved.
- **One post per run.** A `review` run file marks a post, and a retry with that file present posts nothing, as on tools main. Test: a second post of one run sends nothing.
- **Comment only.** The poster sends a review with the event `COMMENT`, carrying the run identifier, the model provider, the exact head, the verdict, and the findings, as ADR 0004 lists them. No code path in `tbhbdev review` sends `APPROVE` or `REQUEST_CHANGES`, an ast-grep rule bans those event names in the review commands, and a test drives every post path through a fake transport that fails on any other event.
- **Approval stays independent.** Approval belongs to the independent approver of ADR 0009, through `tbhb-approve` until the M5 switch decides otherwise. Wave 1 builds no approval path.
- **The host holds the posting token.** In a review run, the run host holds the run-scoped posting token, and the review run's own process never sees it. The run host starts `tbhbdev review post --from-run <run>` as its own child and passes the token on an inherited descriptor that the review run's sandbox never had. Test: the review run's environment and descriptors hold no token, and the poster child reads exactly one.

Two paths post, under two identities, and the poster's wiring picks the token source from the path, never from a flag a caller can set:

| Path | Who runs it | Identity it posts as | Token source |
| --- | --- | --- | --- |
| The run-host helper | The run host, after a review run ends | The reviews identity that `credentials.py` scopes to the run (existing) | The inherited descriptor only. With no descriptor, `--from-run` exits 2 and posts nothing |
| The manual command, `tbhbdev review post <number>` | An interactive session under tbhb or tbhb-agent | The caller's own identity. The Python command uses the GitHub CLI's credentials through `GhApi` | The caller's credential path only. It never opens a run's descriptor or reads a run's token file |

Tests: each path posts under its own identity through a fake transport that records the token, the manual command with a run's descriptor open still uses the caller's token, and `--from-run` from such a session with no descriptor exits 2.

## Slices

The size gate allows 400 added code lines per pull request, tests included. With Python tests at about 2.5 times the source, one slice is about 120 to 150 Python source lines with their tests, and fewer once Go's length is measured.

### Go scaffolding

About six pull requests in tbhb-dev/tbhbdev under `factory/cli`, one after another. The monorepo session owns the repository's own general scaffolding, which these build on. The first Go slice, tools#418, waits for the move and is rebuilt there rather than merged into tools.

1. The front with `factory/cli/internal/dispatch`.
2. The stamp and the bundle test.
3. The cobra root wiring and `internal/config/io/load` with koanf ([appendix K](K-go-stack.md)).
4. The checks with their planted fixtures, the stack rules included.
5. `internal/schemas`, with byte-checked copies of the Configure Unify Execute (CUE) files, in-process validation, and the CUE boundary checks.
6. `internal/ghapi`, with the go-github client, its rules, and a fake transport for tests.

Pull request 1 changes the front and dispatch, so it goes to tbhb before merge (D19). Review and CI come first, a reviewer's verdict is a GO or HOLD comment only, and coordination then puts the pull request on Open. Nobody approves it or runs `tbhb-approve` on it until tbhb says go. Pull requests 2 through 6 go through the gate only if they change a gated path. The phase 1 Python release adds the protocol schema and the `protocol-version` job in tools before the CLI import. The rules ast-grep macro runs beside the scaffolding.

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

The launcher applies host load and memory limits when admitting runs (tools#440).

### Why parallel slices do not collide

- Each slice adds new package directories, and gazelle writes each package's own `BUILD.bazel`, so two slices never edit one build file.
- The arch-lint, depguard, and ast-grep configs match by glob, so a new package needs no config edit.
- A subcommand slice adds one `AddCommand` line to its command group's constructor, the only line two slices share, and a rebase settles it.
- Only the serial steps and dependency bumps touch `go.mod`, `MODULE.bazel`, and the lint configs, one at a time.
- Each slice ports the Python tests for its module as Go table tests in the same pull request, so the Python suite stays the behavior spec until the module's Python code goes.

### Order inside a module

1. Pure types and functions, with their table tests.
2. The adapter, with its `io_go_test` against real tools in a temporary root.
3. The `io/app` wiring and the command, behind its final name.
4. The corpus or golden comparison against the Python command built from the same monorepo commit.
