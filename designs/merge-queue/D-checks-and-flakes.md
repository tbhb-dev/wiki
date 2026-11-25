---
type: Design Appendix
title: In-house merge queue checks and flakes
description: How the queue avoids rerunning what a pull request already proved, retries flakes, and times each step.
tags: [merge-queue, bazel, ci, flakes, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words rdeps -->

# In-house merge queue checks and flakes

Back to [the design](README.md).

## The problem today

In tools a queue pass takes about 1400 seconds, and almost all of it is one Bazel target, `//:pytest_test`, which normally takes about 300 seconds (the brief says; the likely cause is a timed-out first attempt plus a retry). Rerunning failed jobs and the title check also needed manual steps (the brief says).

Two causes stand out, and only one is the queue's to fix:

- **One large test target.** Any Python change invalidates the single pytest target, so Bazel reruns every test. Splitting it per module is tools work, outside this design, and belongs in the tools tracker as tech debt.
- **No reuse between the pull request run and the queue run.** The queue can fix this, in three layers.

## Proven-tree skip

A Git tree SHA names exact content. If two commits have the same tree, a hermetic CI run on one proves the other.

1. When CI passes on any commit the queue watches, it records `(repository, tree SHA, workflow, conclusion)`. That covers candidates and each pull request's `pull_request` run. The run's merge ref, `refs/pull/N/merge`, is main at that time merged with the head.
2. Before pushing a candidate, the queue looks up its tree. CI is skipped only when every workflow the repository requires on candidates has a green record for that tree and is on the reuse list below. Any other required workflow still runs on the candidate.
3. The common case is a one-entry group when main hasn't moved since the pull request's run. Its candidate tree equals the merge ref's tree.

This holds only for checks that depend on content alone. A job that reads the network, the date, or GitHub's API isn't proven by a tree match, so each repository lists in config which workflows the skip may reuse (new). Which base a `pull_request` run used is readable from the run's merge ref commit parents (unverified: the earlier design marks it unverified too).

## The remote cache

Bazel already skips any action whose inputs didn't change, and it reuses cached test results, when the remote cache holds them (existing: bazel-r2-cache). Today, pull request runs write only their own cache namespace with a pull request token, and pushes to main and `merge_group` runs use the main token in a main-only `cache` environment (existing: tools `ci.yml`).

In the first cut, candidates use the pull request token. Their branches are queue-only from the first push (D3, D16), and plan step 5 adds `merge-queue/main/*` to the `cache` environment, so candidates get the main token. Then a candidate's results warm the cache for main's next push run and for later candidates.

Whether a pull request run's namespace is readable from a candidate run is unverified. If it is, candidates should read it too, and they'd rerun only what the merge with newer main changed.

## Affected targets

A later step, after the cache layers prove out: compute the Bazel targets affected by the difference between main's tip and the candidate, with `bazel-diff` or a `bazel query rdeps` over the changed files, and test only those (new). Main's own push run still runs `//...`, so a gap in the affected set shows up there and pauses the queue.

This step adds the most risk for the least gain once the cache works, so it waits for timing data that shows the cache isn't enough.

## Flakes

Not every red candidate means a bad pull request. The queue sorts failures before acting:

| Failure | Detected by | Action |
| --- | --- | --- |
| Infrastructure: runner lost, startup failure, cancellation, a timeout | the job's conclusion and the runner's annotations | retry the group once, then remove as usual |
| A test tagged `flaky` in Bazel | the target's tags in the test log artifact | Bazel's own `flaky = True` attempts handle it inside the job, and the queue records the attempts |
| Any other test failure | the failed targets in the test log artifact | no retry. Split the group and remove the bad entry |

Every retry and flaky attempt goes into the flake table and the `merge_queue_flake_total` metric by repository and target (new). A target that flakes 3 times in 7 days shows on the dashboard's flake panel, and the merge queue maintainer files it as tech debt in that repository's tracker (new). The App gets no issues permission for this. The queue never adds `flaky` tags itself.

This replaces reruns of failed jobs by hand. The queue retries by pushing the same candidate tree as a fresh commit, which needs no `actions: write` permission (new).

## Timing

For each candidate run, the queue reads the workflow run's jobs and their steps, with start and end times, once the run completes (new). It records:

- `merge_queue_ci_step_seconds{repo, workflow, job, step}`, a histogram in Mimir
- per-target test durations from the uploaded `test.xml` files, when the workflow uploads them, as Loki log lines rather than metrics, to keep label cardinality low
- `merge_queue_wait_seconds{repo, priority}` from enqueue to merge, and `merge_queue_group_seconds{repo}` from candidate push to verdict

A Grafana panel shows the slowest steps per repository over 7 days. With it, the 1400-second tools pass would have shown its one slow target directly.
