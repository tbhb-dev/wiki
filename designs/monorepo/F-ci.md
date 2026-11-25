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

<!-- cspell:words monorepo lcov zizmor bzlmod Buildbarn rdeps -->

# One public monorepo for the factory CI

Back to [the design](README.md).

## One workflow

`ci.yml` runs on `pull_request`, `merge_group`, and pushes to `main`. It replaces the `ci.yml` in each repository and the reusable `repo-checks` workflow for everything inside the monorepo.

| Job | What it does | Needs |
| --- | --- | --- |
| `plan` | Computes the affected targets and splits them into shards | none |
| `test` | One matrix entry per shard: `bazelisk test --config=ci --target_pattern_file=<shard>` | `plan` |
| `coverage` | For each affected project, runs coverage over all of the project's test targets and checks the combined figure | `plan` |
| `repo` | Gazelle diff, `bazel mod tidy`, gitleaks over the pull request's commits, commit identity, pull request size, workflow lint, the generated `CODEOWNERS`, and the layering test | none |
| `names` | The private-name scan, if D6 is yes | none |
| `rules` | Runs the nested rules workspace's own tests when a file under `factory/build/rules` changed, if plan step 2 shows the root can't run them | `plan` |
| `conclusion` | Fails unless every job above passed or was skipped on purpose | all |

The main ruleset requires only `conclusion`, pinned to GitHub Actions, as every repository's ruleset does today (existing).

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

A fork gets no secrets on a public repository (per docs), so its run is uncached. Agents push branches inside the repository, so their runs are cached. The merge queue design recorded an operator decision to give queue candidates the `main` token, and that decision covered the in-house queue's branches only. This design leaves GitHub's queue candidates on the `pr` role, and plan step 8 measures whether that costs enough to ask again.

## Coverage

Today the Python macro runs one test target per repository and fails it under 100% branch coverage (existing). With one target per package, no single target sees the whole figure.

A new rule in the rules' Python and Go modules takes a project's test targets, merges their coverage reports, and fails under 100% branch coverage (new). "Project" replaces "repository" as the unit: the folder that holds an `OWNERS` file and a language manifest. When any target in a project is affected, the `coverage` job runs that project's whole set, and unaffected tests come back from the cache with their reports.

How the Go repositories measure branch coverage today wasn't read in this pass (unverified). The Go port's design owns that question.

## Runner sizing

- **Standard hosted runners for everything public.** GitHub doesn't bill them on public repositories (per docs). The workflow uses the same arm64 Ubuntu label the wiki uses today.
- **No larger runners.** They bill on every repository (per docs), so they'd need the operator's approval.
- **No self-hosted runner on `pull_request`.** A public repository runs code from anyone who opens a pull request, and the CI runner hosts hold credentials and network access. The private runner groups don't accept public repositories today anyway (existing).
- **Shards instead of size.** The `plan` job splits targets across up to eight matrix entries by project. A small change gets one entry.
- **The companion keeps the private runner label.** Its jobs are few: schema validation and deploys.

The first sizing target is that a typical pull request's `conclusion` arrives no later than it does in tools today. Plan step 1 records that figure, and step 9 compares after each import.

## Where remote execution would come in

Remote execution sends each build action to a pool of workers instead of running it on the CI runner. It helps when one runner can't finish even with a warm cache, and the likely case is a rules change that reruns every test in the repository.

The order of remedies is fixed, cheapest first:

1. The remote cache, which exists.
2. More shards on free hosted runners.
3. Smaller targets, so less reruns on a change.
4. Remote execution.

Step 4 starts only when full runs stay over 30 minutes of wall time for two weeks with the first three in place.

| Option | Cost | Limit |
| --- | --- | --- |
| A self-hosted executor pool, such as Buildbarn, on owned hosts | Host capacity, and an ops service to run and patch | Runs code from pull requests on owned hosts, so it serves `main` and queue candidates only unless the workers are sandboxed |
| A paid service | A subscription, priced by usage | No paid service starts without the operator's approval |

Neither is in this design's plan. The design only keeps the door open: every action already runs in Bazel's sandbox with declared inputs, which is what remote execution needs.

## The merge queue

The main ruleset gains the merge queue rule in plan step 8, once affected-target CI is the required path.

| Setting | Value | Why |
| --- | --- | --- |
| Merge method | Merge commit | The merge-commit record (existing) |
| Grouping | All entries in a group must pass | Tools' setting today (existing) |
| Group size | 1 to 5, with a wait of 0 minutes | A group forms from what's waiting, and nothing waits for a group to fill |
| Build concurrency | 5 | GitHub tests up to five candidates at once, each on the ones ahead of it |
| Check timeout | 60 minutes | Tools' setting today |

The queue has to carry about 61 landings a day, and more at peaks (proven count). A serial queue at tools' 1,400 seconds a pass can't: a day holds 61 such passes with no slack. Affected-target passes, groups, and five candidates at once are what make it fit, and plan step 9 measures queue wait after each import and stops the imports if the wait grows.

Agents keep one merge path, `tbhbdev alpha merge`, which adds a pull request to the queue (existing decision). Two changes in different projects no longer conflict in practice, but they do share one queue, so a failing candidate delays the ones grouped with it.

The companion has no queue. Its ruleset requires the branch to contain `main`'s tip before a merge, so the checks always ran on what lands. With few changes a day, a rebase race is rare.

## What stays for standalone repositories

The four open source tools and any repository D1 leaves out still call the reusable workflows and composite actions. A reusable workflow must sit in the root `.github/workflows` folder (per docs), so the rules' reusable workflows move there from the imported `factory/build/rules/.github`. Standalone callers change their pin from the rules repository to the monorepo, and composite actions keep working from any folder.
