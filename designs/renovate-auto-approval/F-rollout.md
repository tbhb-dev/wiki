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

## Before enabling a caller

In each calling repository, the administrator installs the App and creates the tier 2 `pins-approver` environment. Allow only the default branch ref. Leave required reviewers unset and store `PIN_APPROVER_CLIENT_ID` and `PIN_APPROVER_PRIVATE_KEY` only in that environment. [Setup steps](D-workflow.md#what-the-administrator-sets-up)

The reusable workflow declares both secrets under `on.workflow_call.secrets`, and the caller maps them by name. The approving job declares `environment: pins-approver` in both modes. Enable the caller after the spike confirms that the job receives both secrets through the environment. [Workflow interface](D-workflow.md#the-workflow_call-interface)

## Evaluate mode

In `evaluate` mode the job runs through the final pre-post validation and then records `would_approve: true` or `false` with the reason code. It requests `permission-pull-requests: read`.

Hand approval goes on as before during this period. The log shows what the workflow would have done beside what people did.

## The first repository

tbhb-dev/rules goes first. It hosts the workflow, it's public, its ruleset already requires one approval, dismisses stale approvals on push, and requires last-push approval ([appendix A](A-evidence.md)), and its Renovate traffic covers every pin kind: action pins, `mise.toml`, `MODULE.bazel`, `.bzl` tool archives, and the lockfiles. A repository with no required approval, such as this wiki, gains nothing from the workflow and isn't a candidate.

The evaluate period runs at least 14 days and covers at least 10 Renovate pull requests, including at least two daily `pins` groups and one `bazel modules` group. If 14 days pass with fewer, it runs until it has them.

## Reviewing the decision log

The rules maintainer collects every `decision.json` from the period with `gh run download` and writes a review as a pull request in tbhb-dev/rules. A second reviewer, from a different model family under the [model roster](../../models/README.md), checks it. The review covers:

- **Every would-approve.** For each, the reviewer reads the pull request's whole diff at H and confirms it changes only pins. One wrong would-approve stops the rollout. The matcher gets a fix and a test, and the evaluate period starts again.
- **Every refusal.** Each refusal has a reason that matches the diff. A refusal of a diff a person judges pin-only becomes either a matcher change, with tests, or a documented limit.
- **Snapshot binding.** Each decision used the head and CI run attempt from its event. The log records which pull request reads ran and whether the base still matched.
- **Coverage.** Each pin kind appears at least once among the decisions, or the review says which kind it hasn't seen.
- **Noise.** The share of runs that end in `checks_pending`, `rate_limited`, or `api_error`, and whether any of them hid a decision.

The repository moves to `enforce` in a pull request that links the review. That pull request is tier 2, because it changes the merge path.

## Enforcing and widening

After tbhb-dev/rules enforces, repositories with one CI workflow reporting all required checks can start their own `evaluate` period. Each uses the same criteria and reviews its results before moving to `enforce`. An ordinary pull request can switch a repository back to `evaluate` or remove its caller at any time.

To stop new approvals across repositories, uninstall the App or revoke its key. Remove both secrets from `pins-approver` in each calling repository when retiring the workflow. Past approvals stay bound to the commits they reviewed.

## Follow-up

After the first repository's evaluate period, the rules maintainer brings the proposed expansion to the operator for review. The expansion would accept renovate-config preset pins in `.github/renovate.json5`. Use the decision log to show which `pins` groups the matcher refused for that file and which would qualify if the preset pins passed the same source default branch check.

Adding the file widens what the workflow can approve and requires a separate operator decision. [Pilot scope](decisions.md)

## The two tier 2 reviews

The module, the reusable workflow, each caller, the App, and the environment configuration change the review and merge path. Every pull request that adds or changes them is tier 2 and gets two fresh, independent review runs. This design also requires two independent reviews of the revised head, including one by Fable. [Architecture Decision Record (ADR) 0001](../../adrs/0001-the-changed-paths-and-content-set-the-review-tier.md)

**Review one, the trust boundary**, reads the workflow, `gather.py`, and `approve.py`:

- The caller uses the default branch's workflow definition. The job never checks out or executes pull request code. It skips pull request artifacts and reads event fields from `GITHUB_EVENT_PATH`.
- The job declares `environment: pins-approver`. Both credentials live only in that environment, and the caller maps the secrets by name. Confirm the spike shows that GitHub denies the job on a non-default run ref before it can read either secret.
- `permissions: {}` holds at the workflow and job level, and the token's scope matches [appendix D](D-workflow.md#minting-the-token).
- The job verifies the reusable workflow checkout before requesting the token. Both modes launch the isolated entry point from that checkout.
- The run takes H from the event and checks the head and base again before posting with `commit_id: H`. It confirms the pull request still qualifies. The third read and dismissal match appendix D.
- Each required check comes from the expected App and workflow. The triggering CI run and attempt identify which result to use for the pull request. Pending, skipped, neutral, and missing checks refuse.
- An ambiguous review write goes through reconciliation before any retry, and unresolved outcomes appear in the decision log.

**Review two, the matcher**, reads `decide.py`, `matcher.py`, `versions.py`, and the tests:

- Each grammar accepts exactly the shapes in [appendix C](C-matcher.md), and the template comparison covers every byte outside the value slots.
- Each reason code has a test, and the meta-test that enforces it runs.
- Each pin kind that names a tag, ancestry, or digest fact refuses when that fact is absent.
- The lockfile comparison refuses anything outside its listed entries.
- The reviewer tries to write a diff that isn't pin-only and still passes, and adds it as a test whether or not it passes.
