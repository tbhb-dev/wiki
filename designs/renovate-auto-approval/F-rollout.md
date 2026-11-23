---
type: Design Appendix
title: Renovate auto-approval rollout
description: Evaluate mode, the first repository, the decision-log review, and what the two tier 2 reviews check.
tags: [renovate, approvals, rollout, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Renovate auto-approval rollout

Back to [the design](README.md).

## Evaluate mode

In `evaluate` mode the run does everything enforce mode does, up to the second head read, and then stops. It never mints a token with write scope for a post it won't make, so evaluate mode asks for `permission-pull-requests: read` instead of write. The summary and `decision.json` record `would_approve: true` or `false` with the reason code.

Hand approval goes on as before during this period. The log shows what the workflow would have done beside what people did.

## The first repository

tbhb-dev/rules goes first. It hosts the workflow, it's public, its ruleset already requires one approval, dismisses stale approvals on push, and requires last-push approval ([appendix A](A-evidence.md)), and its Renovate traffic covers every pin kind: action pins, `mise.toml`, `MODULE.bazel`, `.bzl` tool archives, and the lockfiles. A repository with no required approval, such as this wiki, gains nothing from the workflow and isn't a candidate.

The evaluate period runs at least 14 days and covers at least 10 Renovate pull requests, including at least two daily `pins` groups and one `bazel modules` group. If 14 days pass with fewer, it runs until it has them.

## Reviewing the decision log

The rules maintainer collects every `decision.json` from the period with `gh run download` and writes a review as a pull request in tbhb-dev/rules. A second reviewer, from a different model family under the [model roster](../../models/README.md), checks it. The review covers:

- **Every would-approve.** For each, the reviewer reads the pull request's whole diff at H and confirms it changes only pins. One wrong would-approve stops the rollout. The matcher gets a fix and a test, and the evaluate period starts again.
- **Every refusal.** Each refusal has a reason that matches the diff. A refusal of a diff a person judges pin-only becomes either a matcher change, with tests, or a documented limit.
- **Head binding.** No decision used a head other than the one in its event, and the timings of the three reads look sane.
- **Coverage.** Each pin kind appears at least once among the decisions, or the review says which kind it hasn't seen.
- **Noise.** The share of runs that end in `checks_pending`, `rate_limited`, or `api_error`, and whether any of them hid a decision.

The repository moves to `enforce` in a pull request that links the review. That pull request is tier 2, because it changes the merge path.

## Enforcing and widening

After tbhb-dev/rules enforces, each other repository adds its caller in `evaluate`, runs its own period with the same bar, and moves to `enforce` with its own review. A repository can drop back to `evaluate`, or remove its caller, at any time in an ordinary pull request.

The stop for every repository at once is to uninstall the App or revoke its key. Either takes effect on the next run, and nobody needs to undo a review, because each approval binds to a head that has already merged or that the next push dismisses.

## The two tier 2 reviews

The module, the reusable workflow, each caller, and the App change the review and merge path, so each pull request that adds or changes them is tier 2 under [Architecture Decision Record (ADR) 0001](../../adrs/0001-the-changed-paths-and-content-set-the-review-tier.md) and gets two fresh review runs. The two reviews of the build split their focus so that one isn't anchored on the other.

**Review one, the trust boundary**, reads the workflow, `gather.py`, and `approve.py`:

- No step checks out, downloads, or runs anything from the pull request, and no `${{ }}` expression carries a pull request field into a script.
- `permissions: {}` holds at the workflow and job level, and the token's scope matches [appendix D](D-workflow.md#minting-the-token).
- The run reads H once, every ref passed to the API is H or the merge base, and the three head reads and the dismissal path match appendix D.
- Each required check binds to its `integration_id`, and pending, skipped, neutral, and missing checks refuse.
- Every exception path ends in a reason code and no approval.

**Review two, the matcher**, reads `decide.py`, `matcher.py`, `versions.py`, and the tests:

- Each grammar accepts exactly the shapes in [appendix C](C-matcher.md), and the template comparison covers every byte outside the value slots.
- Each reason code has a test, and the meta-test that enforces it runs.
- Each pin kind that names a tag, ancestry, or digest fact refuses when that fact is absent.
- The lockfile comparison refuses anything outside its listed entries.
- The reviewer tries to write a diff that isn't pin-only and still passes, and adds it as a test whether or not it passes.
