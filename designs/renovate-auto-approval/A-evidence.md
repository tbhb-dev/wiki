---
type: Design Appendix
title: Renovate auto-approval evidence
description: What the repositories, rulesets, Renovate presets, and GitHub documentation show for the Renovate auto-approval design.
tags: [renovate, approvals, evidence, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Renovate auto-approval evidence

Back to [the design](README.md).

Each claim carries one label. **Proven** means a spike or a merged pull request showed it. **Existing** means a current file or GitHub's documentation says it, read on 2026-11-23. **New** means this design proposes it. **Unverified** means nobody has checked it yet, and the plan gives the step that checks it.

Only the environment probe has run for this design (2026-11-24, setup below), so only its two rows say proven. Plan step 2's remaining spike turns the other unverified rows into proven or refuted ones.

## The repositories and presets

| Claim | Label | Source |
| --- | --- | --- |
| Self-hosted Renovate runs as an org GitHub App, with a client ID and private key held as org secrets | existing | `.github/workflows/renovate.yaml` in this repository |
| The Renovate workflow mints its token with `actions/create-github-app-token` pinned by commit, passing `client-id` and `private-key` | existing | `.github/workflows/renovate.yaml` in tbhb-dev/renovate-config |
| Renovate's token holds checks write and statuses write, among others | existing | the same workflow's `permission-*` inputs |
| Renovate commits through the API with `platformCommit: "enabled"`, so GitHub signs them as the App | existing | `default.json5` in tbhb-dev/renovate-config |
| Renovate enables GitHub's auto-merge and merges only after the ruleset's approval and checks | existing | `default.json5`, and [Architecture Decision Record (ADR) 0024](../../adrs/0024-renovate-pull-requests-automerge-after-approval-and-ci.md) |
| Renovate rebases only on conflict, `rebaseWhen: "conflicted"` | existing | `default.json5` |
| The daily `pins` group holds action pins, mise versions, renovate-config pins, and the tbhb-dev/rules pins, and major updates get their own pull request | existing | `default.json5` and `bazel.json5` in tbhb-dev/renovate-config |
| `bazel_dep` updates form a `bazel modules` group that runs `bazel mod deps` to refresh `MODULE.bazel.lock` | existing | `bazel.json5` |
| The `bazel` manager rewrites the URL version, `strip_prefix`, and `sha256` of tool archives | existing | `bazel.json5` |
| Renovate updates URL versions and digests for `http_archive` calls indented inside a function | existing | [tbhb-dev/rules#376](https://github.com/tbhb-dev/rules/pull/376/files) |
| This repository pins tbhb-dev/rules as a `git_override` commit in `MODULE.bazel` and as `@<commit>  # main` in workflow `uses:` lines | existing | `MODULE.bazel` and `.github/workflows/ci.yml` in this repository |
| CI runs Bazel with `--lockfile_mode=error` | existing | `.bazelrc` and `preset.bazelrc` in this repository |
| CI runs without invoking mise or validating the committed `mise.lock` | existing | `mise.toml` and `.github/workflows/ci.yml` in this repository |
| tbhb-dev/rules' ruleset requires one approval, dismisses stale approvals on push, requires last-push approval, and requires the `conclusion` check from integration 15368 | existing | `.github/rulesets/main.json` in tbhb-dev/rules |
| This repository's ruleset requires no approval | existing | `.github/rulesets/main.json` in this repository |
| zizmor here requires every action pinned by commit | existing | `.github/zizmor.yml` in this repository |
| Repository checks in tbhb-dev/rules keep Python modules and their `test_*.py` files under `lint/private/` and `repo/` | existing | the tbhb-dev/rules tree |

## The decision records

| Claim | Label | Source |
| --- | --- | --- |
| A pin to a reviewed and merged first-party commit gets one review run, and third-party version and checksum updates need only CI | existing | [ADR 0001](../../adrs/0001-the-changed-paths-and-content-set-the-review-tier.md) |
| Changes to the review and merge path, credentials, and permissions are tier 2 | existing | ADR 0001 |
| An approval requires the `conclusion` check and verified signatures on every commit | existing | [ADR 0009](../../adrs/0009-the-reviews-app-posts-approvals-for-an-independent-approver.md) |
| An update that runs a command with Renovate's token keeps its changes to manifests and lockfiles | existing | ADR 0024 |

The operator's decision of 2026-11-23 governs qualifying pin-only Renovate pull requests. The App's approval replaces the first-party review run when the workflow proves that the source's default branch contains the new commit. Plan step 1 amends the review rules to record the exception. [Decision record](decisions.md#d2-does-the-workflows-approval-replace-the-one-review-run-that-a-first-party-pin-gets-today)

## GitHub's documented behavior

| Claim | Label | Source |
| --- | --- | --- |
| `check_suite` and `check_run` don't trigger workflows when GitHub Actions created the check suite | existing | GitHub Docs, "Events that trigger workflows" |
| `workflow_run` runs only from the default branch's workflow file, can read secrets, and chains at most three levels | existing | the same page |
| The tier 2 environment policy checks the run ref in the calling repository | existing | GitHub Docs, "Deployments and environments" |
| With named secret mappings in the caller, an approving job can read its environment secrets on the default branch ref | proven | A `workflow_run` caller on the default branch passed both secrets by name to a cross-repository reusable job. The job log shows both secrets present at length 48 each, with 12-character SHA-256 prefixes equal to the ones recorded when the secrets were set. No approval wait occurred. The run records no automated comparison: run 45443450705, job 138620660924 in tbhb-dev/pins-probe-caller |
| GitHub denies a job on the non-default refs `agent/probe-2a` and `agent/probe-2b` that requests `pins-approver`, with no runner assigned and no step run, so no secret is available. A run from a pull request branch is not tested | proven for those two refs | Run 45443595028 (job 138621235602, an edited reusable-call caller on `agent/probe-2a`) and run 45443608258 (job 138621258121, a job declaring `environment: pins-approver` directly, on `agent/probe-2b`) in tbhb-dev/pins-probe-caller. Both ran through `workflow_dispatch` on a branch with no pull request. Each job record shows no runner assigned (`runner_id` 0, empty `runner_name`), an empty steps list, and the annotation "Branch ... is not allowed to deploy to pins-approver due to environment protection rules." The control, CI run 45443403838, succeeded on the same runner label, `ubuntu-26.04-arm`, with an assigned runner and three steps |
| `pull_request_target` runs with secrets, and running untrusted code under it can lead to vulnerabilities | existing | the same page |
| The branch rules endpoint returns every active rule from repository and organization rulesets, and leaves out rulesets in `evaluate` or `disabled` | existing | GitHub REST docs, "Rules for repositories" |
| The branch rules endpoint's required permission isn't documented | unverified | the same page, which names none. Plan step 2 tests it |
| A review's `commit_id` names the commit it reviews, and defaults to the latest | existing | GitHub REST docs, "Pull request reviews" |
| On a protected branch, only admins and the dismissal allowlist can dismiss a review | existing | the same page |
| Required approvals count only from reviewers with write permission | existing | GitHub Docs, "About protected branches" |
| Stale-approval dismissal fires on pushes that change the pull request's diff | existing | the same page |
| Last-push approval requires someone other than the pusher to approve the latest reviewable push | existing | the same page |
| GitHub lets `skipped` and `neutral` conclusions satisfy a required check | unverified | widely reported, and plan step 2 records it |
| An installation token expires after one hour | existing | GitHub Docs, "Authenticating as a GitHub App installation" |
| `job.workflow_repository` and `job.workflow_sha` identify the repository and commit of the workflow defining the current job | existing | [GitHub Actions contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts#job-context) |
| The Actions API exposes workflow runs and jobs for a specific run attempt, including check-run URLs, and allows unauthenticated reads of public data | existing | [Workflow runs](https://docs.github.com/en/rest/actions/workflow-runs) and [workflow jobs](https://docs.github.com/en/rest/actions/workflow-jobs) |
| The Checks API exposes a check run by ID and reports its suite and head commit | existing | [Check runs](https://docs.github.com/en/rest/checks/runs) |
| GitHub release asset metadata includes the asset's browser download URL | existing | [Get a release asset](https://docs.github.com/en/rest/releases/assets#get-a-release-asset) |

## The environment probe

Run on 2026-11-24 in two scratch repositories, tbhb-dev/pins-probe-caller and tbhb-dev/pins-probe-callee. Both are public, call across repositories, and have no rulesets. The `pins-approver` environment used a custom branch policy with one rule, `main`, no required reviewers, and no tag rules. Only dummy secrets were stored. The callee stub set `permissions: {}` at workflow and job level.

Untested: the probe saw only `main` allowed and `agent/probe-2a` and `agent/probe-2b` denied. A run dispatched from a pull request branch, other branch names, tag refs, and a fork were not tested.

## Open questions for the spike

Plan step 2 answers these in a scratch repository with a copy of the tbhb-dev/rules ruleset:

- Whether the environment policy also holds for a run dispatched from a pull request branch, other branch names, tag refs, and a fork. [The environment probe](#the-environment-probe) proved `main` allowed and `agent/probe-2a` and `agent/probe-2b` denied only.
- Whether GitHub counts the App's approval with pull requests write, contents read, checks read, and mandatory metadata read. If counting the review requires a wider grant, take the result to the operator under [D3](decisions.md).
- Whether the App can dismiss its own review when the ruleset restricts dismissals, and whether the `dismissal_failed` path then leaves stale dismissal as the only guard.
- Whether a review posted with an older `commit_id` after a push counts toward the requirement.
- How GitHub attributes check suites and check runs across reruns. Record the workflow run ID, attempt, job check-run IDs, suite ID, and suite head commit, including when CI tests a merge commit.
- What associations the event and Actions API return when two or more pull requests share H. Confirm that the workflow refuses another pull request's CI and refuses ambiguous associations.
- Whether GitHub dismisses or still counts a review after a target branch change that leaves H unchanged. Change the branch between the final pre-post read and the approval, and check the effect of the dismissal call.
- Whether the reusable job can check out its exact public workflow revision and upload its decision artifact with `permissions: {}` at the caller, workflow, and job levels.
- Whether the App needs Actions read to gather the required workflow run and job evidence. Test the public pilot repository without that grant.
- Which permission the branch rules endpoint needs.
- Whether GitHub counts `skipped` and `neutral` conclusions toward required checks.
- The author, committer, and `verification` fields of a Renovate `platformCommit` commit, and whether GitHub fills `performed_via_github_app` on Renovate's pull requests.
- Whether a fetch of two commits without blobs, followed by `git diff`, works with the installation token on a hosted runner.
