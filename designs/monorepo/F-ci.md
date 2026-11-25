---
type: Design Appendix
title: One public monorepo for the factory CI
description: The monorepo's one CI workflow on GitHub Actions, with affected targets, the remote cache, coverage, runner sizing, remote execution, and the merge queue.
tags: [monorepo, ci, bazel, github-actions, merge-queue, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo lcov zizmor bzlmod Buildbarn rdeps cutover -->

# One public monorepo for the factory CI

Back to [the design](README.md).

## One workflow

`ci.yml` runs on `pull_request`, `merge_group`, and pushes to `main`. It replaces the `ci.yml` in each repository, and its jobs are written inline, so it calls no workflow or action in an old repository (D27).

| Job | What it does | Needs |
| --- | --- | --- |
| `plan` | Computes the affected targets and splits them into shards | none |
| `test` | One matrix entry per shard: `bazelisk test --config=ci --target_pattern_file=<shard>` | `plan` |
| `coverage` | For each affected project, runs coverage over all of the project's test targets and checks the combined figure | `plan` |
| `repo` | Gazelle diff, `bazel mod tidy`, gitleaks over the pull request's commits, commit identity, pull request size, the pull request body check, workflow lint, and the layering test | none |
| `names` | The private-name scan, if D6 is yes | none |
| `import` | On an import pull request, checks the copy against its source and its path map ([appendix G](G-import-and-cutover.md#importing-one-repository)) | none |
| `rules` | Runs the nested rules workspace's own tests when a file under `factory/rules` changed, if plan step 2 shows the root can't run them | `plan` |
| `conclusion` | Fails unless every job above passed or was skipped on purpose | all |

The main ruleset requires one check, `conclusion`, pinned to GitHub Actions as in every repository's ruleset today (existing). It also requires an independent approval of the exact head: one approving review that isn't the author's or the last pusher's, dismissed by any new commit. The review tool's App posts that approval only after the fresh review runs the change's tier requires ([appendix C](C-layout-and-ownership.md#mapping-apps-rulesets-and-branch-rules), new). A green `conclusion` alone therefore can't merge a change.

No workflow uses a `paths` filter. GitHub reads at most 300 changed files for one (per docs), so an import or a lock update could skip a job that should run. The `plan` job decides from the build graph instead.

## Affected targets

`bazel-diff` writes one file of target hashes for a commit. A hash covers a target's own attributes, its source files, and the hashes of everything it depends on. Two files compared give the targets whose hashes differ.

1. A push to `main` generates the hash file for that commit and saves it, keyed by the commit hash.
2. The `plan` job on a pull request loads the file for the base commit. If it's missing, the job checks out the base and generates it.
3. The job generates the file for the head and compares the two.
4. It keeps the test targets from the result, groups them by project, and writes the shard files.

A change to a root file that every target reads, such as `MODULE.bazel`, a lock file, or `.bazelrc`, changes every hash, and the job then plans a full run. So does a change to the rules, because every build file loads them.

A push to `main` runs `//...` with no plan. That run is the safety net for a mistake in the plan, and it fills the remote cache that the next pull requests read. If it fails after the queue passed the same commit, the failure opens an incident, because it means the plan missed a target.

Whether `bazel-diff` handles nested local modules under bzlmod is unverified, and plan step 2 tests it.

## The remote cache

The shared remote cache and its token roles stay as they are (existing).

| Event | Token role | Reads | Writes |
| --- | --- | --- | --- |
| Push to `main` | `main`, through the main-only `cache` environment | main | main |
| `merge_group` | `pr` | its own namespace, then main | its own namespace |
| `pull_request` from a branch in the repository | `pr` | its own namespace, then main | its own namespace |
| `pull_request` from a fork | none | nothing | nothing |

A fork gets no secrets on a public repository (per docs), so its run is uncached. Agents push branches inside the repository, so their runs are cached. The merge queue design recorded an operator decision to give queue candidates the `main` token, and that decision covered the in-house queue's branches only. This design leaves GitHub's queue candidates on the `pr` role, and plan step 10 measures whether that costs enough to ask again.

## Coverage

Today the Python macro runs one test target per repository and fails it under 100% branch coverage (existing). With one target per package, no single target sees the whole figure.

A new rule in the rules' Python and Go modules takes a project's test targets, merges their coverage reports, and fails under 100% branch coverage (new). "Project" replaces "repository" as the unit: a direct child of `factory/`, `products/`, `lib/`, or `docs/`. When any target in a project is affected, the `coverage` job runs that project's whole set, and unaffected tests come back from the cache with their reports.

How the Go repositories measure branch coverage today wasn't read in this pass (unverified). The Go port's design owns that question.

## Runner sizing

- **Standard hosted runners for everything public.** GitHub doesn't bill them on public repositories (per docs). The workflow uses the same arm64 Ubuntu label the wiki uses today.
- **Standard macOS runners for Swift.** Only shards that hold Swift targets run there, and they're also free on a public repository (per docs).
- **No larger runners.** They bill on every repository (per docs), so they'd need the operator's approval.
- **No self-hosted runner on `pull_request`.** A public repository runs code from anyone who opens a pull request, and the CI runner hosts hold credentials and network access. The private runner groups don't accept public repositories today anyway (existing).
- **Shards instead of size.** The `plan` job splits targets across up to eight matrix entries by project. A small change gets one entry.
- **The companion keeps the private runner label.** Its jobs are few: schema validation and deploys.

The first sizing target is that a typical pull request's `conclusion` arrives no later than it does in tools today. Plan step 1 records that figure, and step 11 compares after each import.

## Where remote execution would come in

Remote execution sends each build action to a pool of workers instead of running it on the CI runner. It helps when one runner can't finish even with a warm cache, and the likely case is a rules change that reruns every test in the repository.

The order of remedies is fixed, cheapest first: the remote cache, which exists, then more shards on free hosted runners, then smaller targets, and remote execution last. Remote execution starts only when full runs stay over 30 minutes of wall time for two weeks with the first three in place.

| Option | Cost | Limit |
| --- | --- | --- |
| A self-hosted executor pool, such as Buildbarn, on owned hosts | Host capacity, and an ops service to run and patch | Runs code from pull requests on owned hosts, so it serves `main` and queue candidates only unless the workers are sandboxed |
| A paid service | A subscription, priced by usage | No paid service starts without the operator's approval |

Neither is in this design's plan. The design only keeps the door open: every action already runs in Bazel's sandbox with declared inputs, which is what remote execution needs.

## The merge queue

The main ruleset gains the merge queue rule in plan step 10, once affected-target CI is the required path.

| Setting | Value | Why |
| --- | --- | --- |
| Merge method | Merge commit | The merge-commit record (existing) |
| Build concurrency | 5 | CI tests up to five candidates at once, each built on the ones ahead of it |
| Merge limits | A group of 1 to 5, with a wait of 0 minutes | Up to five passed entries merge together, and nothing waits for a group to fill |
| Failing entries | Every entry in a group must pass | Tools' setting today (existing) |
| Check timeout | 60 minutes | Tools' setting today |

Two settings do different jobs. Build concurrency is CI capacity: it sets how many candidates are tested at once. The merge limits act only after checks pass: they set how many passed entries land in one merge.

The queue has to carry about 61 landings a day, and more at peaks (proven count). A serial queue at tools' 1,400 seconds a pass can't, because a day holds 61 such passes with no slack. Shorter affected-target passes and five candidates in flight supply the CI capacity. Plan step 11 measures queue wait after each import and stops the imports if the wait grows.

GitHub recommends about one merge a minute and six pushes a minute for a repository ([appendix A](A-evidence.md#githubs-repository-limits)). Grouped merges keep `main` at one merge for each finished pass, far under the first. For the second, agents push once for each review round, not once for each commit, and `tbhbdev` backs off and retries a refused push.

Agents keep one merge path, `tbhbdev alpha merge`, which adds a pull request to the queue (existing decision). Two changes in different projects no longer conflict in practice, but they do share one queue, so a failing candidate delays the ones grouped with it.

The companion has no queue. Its ruleset requires the branch to contain `main`'s tip before a merge, so the checks always ran on what lands. With few changes a day, a rebase race is rare.

## What stays for standalone repositories

The monorepo calls no reusable workflow, its own included. The four open source tools and any repository D1 leaves out keep calling the reusable workflows and composite actions at the pins they hold, in the archived rules repository, until each one's own cutover.

For that cutover, the rules' reusable workflows are also published from the monorepo's root `.github/workflows` folder, where a reusable workflow must sit (per docs). A standalone repository then changes its pin to the monorepo. Composite actions work from any folder.
