---
type: Design Appendix
title: In-house merge queue evidence
description: What the live reads, repository files, decision records, and the brief show, each labeled proven, existing, new, or unverified.
tags: [merge-queue, evidence, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words ALLGREEN HEADGREEN retarget fnmatch -->

# In-house merge queue evidence

Back to [the design](README.md).

Labels: **proven** means a live read or run on 2026-11-24, **per docs** means docs.github.com as read on 2026-11-24, **existing** means a file or record at the named commit, **new** means this design proposes it, and **unverified** means nobody checked it. "The brief says" marks facts from the task brief that this pass didn't re-read.

## Live reads

| Claim | Label | Source |
| --- | --- | --- |
| ops' main branch rules: deletion, non-fast-forward, required signatures, a pull request rule with one approval, stale dismissal, last-push approval, resolved threads, and merge commits only, plus the `conclusion` check pinned to `integration_id` 15368. The read has no merge queue rule and no linear history rule. It doesn't return bypass actors, so their absence rests on the checked-in files | proven | `gh query repos/tbhb-dev/ops/rules/branches/main` |
| tools' live merge queue rule: method `MERGE`, `ALLGREEN`, build 1, merge 1 to 5, wait 5 minutes, timeout 60 minutes, and merge commits only | proven | `gh query repos/tbhb-dev/tools/rules/branches/main` |
| The org plan is Team. The merge queue rule works on tools, rules, and event-gateway, and on ops and agent-config the API returns HTTP 422, invalid rule `merge_queue` | the brief says | task brief, 2026-11-24 |
| rules' queue runs `MERGE` and `ALLGREEN` with build 1, group 1 to 1, wait 0, timeout 90 | the brief says | task brief |
| A tools queue pass takes about 1400 seconds, mostly `//:pytest_test`, normally about 300 seconds | the brief says | task brief |

## Repository files

| Claim | Label | Source |
| --- | --- | --- |
| tools' checked-in main ruleset still lists linear history and squash, unlike the live rules, so checked-in rulesets aren't the live state | existing | tools `.github/rulesets/main.json` at f342d9c5 |
| tools CI runs on `merge_group`, gives the main-only `cache` environment to `push` and `merge_group` runs, and runs the wheel and release jobs on any `push` event | existing | tools `.github/workflows/ci.yml` at f342d9c5 |
| ops' CI gives the `cache` environment to `push` and `merge_group` runs, and agent-config's to `push` runs | existing | each repository's `.github/workflows/ci.yml` on main, 2026-11-24 |
| `tbhbdev alpha merge` builds the merge message, checks Conventional Commit titles and the trailer paragraph, scans the message with gitleaks, merges with `--match-head-commit`, and prints the message under `--dry-run`. It refuses `BLOCKED` merge states, so a queue-run message read needs a mode that skips that refusal | existing, and new for the mode | tools `src/tbhbdev/alpha/squash_merge.py` at f342d9c5 |
| A `MERGE`-method GitHub queue keeps the subject and body passed to `gh pr merge` | existing | the same file's docstring, citing tools#370 and merge commit 5800789a |
| `tbhbdev alpha gh-app` mints App installation tokens with the key read through Connect | existing | tools `src/tbhbdev/alpha/gh_app.py` at f342d9c5 |
| `tbhbdev alpha loop` validates config and refuses to run live | existing | tools `src/tbhbdev/alpha/loop/__init__.py` at f342d9c5 |
| The event gateway keeps a durable JetStream log and serves `GET /v1/events?after=N` with a `last_seq` cursor and a `gap` object | existing | event-gateway `README.md` at c660b31 |
| The gateway delivers to pluggable targets, each from its own durable consumer | existing | event-gateway `cmd/event-gateway/runner.go` at c660b31 |
| The branch-prefix ruleset restricts creation and updates outside `agent/**` and `renovate/**`, with the operator as the only bypass actor, and the checked-in copy is disabled | existing; live state unverified | rules `.github/rulesets/branch-prefixes.json` |
| The agent trailer ruleset targets `agent/**` branches only | existing | rules `.github/rulesets/agent-trailers.json` |

## Decision records and designs

| Claim | Label | Source |
| --- | --- | --- |
| Pull requests land through merge commits only | existing | [`adrs/0020-pull-requests-land-through-merge-commits.md`](../../adrs/0020-pull-requests-land-through-merge-commits.md) |
| Merge commits carry `Merged-by` | existing | [`adrs/0014-merge-commits-record-the-merger.md`](../../adrs/0014-merge-commits-record-the-merger.md) |
| Stacked pull requests are reviewed against their own base, retarget to main after the parent merges, and keep their review when head and diff don't change | existing | [`adrs/0006-pull-requests-in-one-repository-can-be-reviewed-in-parallel.md`](../../adrs/0006-pull-requests-in-one-repository-can-be-reviewed-in-parallel.md) |
| Authors may merge or rebase main into their branch, and push with a lease | existing | [`adrs/0021-authors-may-update-and-curate-pull-request-branches.md`](../../adrs/0021-authors-may-update-and-curate-pull-request-branches.md) |
| The 15-minute reconcile read, the Unix service account, the `Merged-by` tool form, and no time delays on merges | existing | tbhb-dev/wiki-private `designs/pr-review-merge-queues/`, `decisions.md` |
| The known-bot rule for Renovate, and the `hold` and `urgent` labels | proposed in a design nobody has built | tbhb-dev/wiki-private `designs/pr-review-merge-queues/`, `G-merge-queue.md` |
| The review and merge queue runs as self-hosted Temporal workflows behind the event gateway | existing | the operator's decision of 2026-11-22 |
| Agents merge only through `tbhbdev alpha merge`, and the guard blocks direct API merges for agents | existing | the operator's decision of 2026-11-23 |
| GitHub's queue stays on public repositories, ops and agent-config keep plain required checks, and there's no plan upgrade | existing | the operator's decision of 2026-11-24 |
| rules owns a review of which CI jobs run on pull requests and which in the queue, with tools first | existing, in progress | the operator's decision of 2026-11-24 |
| Each repository's main cache token lives in its main-only `cache` environment | existing | the operator's decision of 2026-11-13 |
| No production Temporal server is deployed | existing | `git grep -il temporal` in ops finds only telemetry and network policy files |
| The gateway plans a Temporal target, and workflows on owned hosts | existing as a plan | [`designs/event-gateway/decisions.md`](../event-gateway/decisions.md), D15 and the workflow rule |

## GitHub behavior

| Claim | Label | Source |
| --- | --- | --- |
| Rulesets with App-pinned required checks work on private repositories under Team | proven | the ops live read |
| The docs don't use the names `ALLGREEN` and `HEADGREEN`. Their setting "Only merge non-failing pull requests", when on, makes every pull request in a group satisfy required checks. When off, a failed pull request can join a group if the last one passes | per docs | GitHub's merge queue docs |
| GitHub's queue names its branches `gh-readonly-queue/{base_branch}/...`, and the merge queue page states no plan requirement | per docs | the same page |
| Environments, environment secrets, and deployment branch policies work on Team private repositories. Wait timers and required reviewers don't | per docs | GitHub's environments docs |
| Rulesets cover Team. A required check can name an App as its expected source, and the restrict-updates rule lets only bypass actors push, which can be teams or Apps | per docs | GitHub's ruleset docs |
| An App is offered as a check's expected source only after it reports the check, which needs `statuses: write` | per docs, as the reviewer cites them | GitHub's ruleset docs, required status checks |
| GitHub can evaluate required checks on the test merge commit instead of the head when that commit has a status | per docs, as the reviewer cites them | GitHub's required-check troubleshooting docs; plan step 3 tests it |
| A ruleset pattern's trailing `**` doesn't cross a slash under GitHub's `fnmatch` matching, so `merge-queue/main/<entry>` needs `refs/heads/merge-queue/**/*` | per docs, as the reviewer cites them | GitHub's ruleset docs; plan step 1 tests it on the real path |
| An App push that changes files under `.github/workflows` needs `workflows: write` | per docs, as the reviewer cites them | GitHub's App permission docs; plan step 1 tests it |
| A workflow rerun keeps its original ref and SHA, and an environment's secrets are read when the job starts | per docs, as the reviewer cites them | GitHub's Actions docs |
| A ruleset's evaluate mode needs Enterprise | per docs, as the reviewer cites them | GitHub's ruleset API docs |
| GitHub-created merge commits pass required signatures | existing | ops requires signatures and merges through `gh pr merge` |
| A push with an App installation token starts workflows, unlike one with `GITHUB_TOKEN` | per docs; unverified live | GitHub's token docs; plan step 1 tests it |
| GitHub verifies an App's commits made through the API with no custom author, committer, or signature details. Whether the Git database API, the contents API, and the merge endpoint each yield signed commits is unsettled | per docs; unsettled | GitHub's signature verification docs; plan step 1 checks the merge endpoint on a real commit |
| Branch restrictions are available on every repository of a Team organization | per docs | GitHub's protected branch docs; not needed under E1 |
| Changing a pull request's base keeps its approval | unverified; the docs don't settle it | plan step 4 tests it |
| A pull request run's cache namespace is readable from another run | unverified | bazel-r2-cache, not read in this pass |
| Hosted runner minutes included with Team, and what candidate runs would add | unverified | GitHub's plan page, not read in this pass |
