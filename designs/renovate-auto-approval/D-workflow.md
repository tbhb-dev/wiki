---
type: Design Appendix
title: Renovate auto-approval workflow
description: The trigger, head binding, author check, required checks, approver App, and workflow interface of the Renovate auto-approval design.
tags: [renovate, approvals, github-actions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Renovate auto-approval workflow

Back to [the design](README.md).

Every GitHub API path below is a REST path under `https://api.github.com`. The claims about GitHub's behavior carry the labels and citations in [appendix A](A-evidence.md).

## The trigger

Each approval needs the App's credentials and the CI results at the pull request head.

| Trigger | When it fires | Secret and code | Tradeoff |
| --- | --- | --- | --- |
| `pull_request_target` | On pull request activity, such as `opened` and `synchronize` | Runs the base branch's workflow | Fires before CI finishes. The job needs to poll or wait for a second event. Checking out and running code from the head can expose secrets |
| `workflow_run` | When a named workflow completes | Runs the default branch's workflow on the default branch ref | Fires when CI's result exists. The approving job treats pull request content as data |
| `check_suite` or `check_run` | When a check suite or check run completes | Runs the default branch's workflow | GitHub doesn't fire either for suites that GitHub Actions created, and every required check here comes from Actions, so neither fires |

The caller uses `workflow_run` with `types: [completed]` and lists exactly one CI workflow. That workflow must report all required checks, and each completion triggers a decision. [D5](decisions.md)

- To inspect pull request changes, the job reads the diff and file text as data through Git objects and the API. It never checks out or executes pull request code. CI artifacts stay outside the job.
- GitHub runs the caller's workflow definition from the default branch. The job runs code from tbhb-dev/rules at the commit that definition pins.
- The approving job declares `environment: pins-approver`. The tier 2 environment policy matches the run ref in the calling repository and doesn't check which workflow file started the run. [The spike](E-failures-and-tests.md#environment-spike) tests whether GitHub admits the job and exposes its secrets only on the default branch ref.
- No pull request field, such as a title or a branch name, reaches a `run:` script through a `${{ }}` expression. The script reads the event payload from `GITHUB_EVENT_PATH` and gets everything else from the API.

zizmor's `dangerous-triggers` audit flags `workflow_run`. Each caller adds an ignore entry for its caller file, with a comment that points here.

## The exact-head binding

The review endpoint accepts a commit ID and has no precondition for the current head or base. The run uses three pull request reads and posts against a specific commit.

1. **Read once.** The run takes H, the head of the pull request, from `workflow_run.head_sha`. It also records the triggering run ID, attempt, and head repository ID. CI can test a merge commit for H.
2. **Find the pull request.** `GET /repos/{owner}/{repo}/commits/{H}/pulls` must return exactly one open pull request whose head repository is this repository and whose head is H. Otherwise the run ends with `pr_not_unique` or `fork_head`.
3. **Read the initial state.** `GET /repos/{owner}/{repo}/pulls/{number}` must show `head.sha` equal to H, or the run ends with `head_mismatch_at_start`. Record the base repository ID and target branch, require the caller repository and its default branch, and compute the merge base of the reported `base.sha` and H.
4. **Gather and decide.** Read commits and diff data by commit ID. Bind the checks to the triggering CI snapshot as described below.
5. **Second read, just before approval.** Recheck the required checks against the same snapshot, then re-read the pull request. Require H, the same base repository and target branch, and the same merge base of the fresh `base.sha` and H. Also require `state` open, `draft` false, and no opt-out label. A changed head ends the run with `head_moved`, and a changed base ends it with `base_changed`. In `evaluate` mode the run stops after recording the decision.
6. **Bound post.** In `enforce` mode send `POST /repos/{owner}/{repo}/pulls/{number}/reviews` with `event: APPROVE` and `commit_id: H`. Require the response's `commit_id` to equal H.
7. **Read after the post.** Read the pull request and compare H, the base repository, target branch, and merge base again. If the head or base changed, call `PUT .../reviews/{review_id}/dismissals` and log `head_moved_after_approval` or `base_changed_after_approval`. The spike tests dismissal and changes to the target branch at the same head.

The reads check for head changes, and the response's `commit_id` confirms the review targets H. A push can land between the second read and the post. The workflow relies on two ruleset settings to guard that interval. `dismiss_stale_reviews_on_push` voids stale approvals on push, and `require_last_push_approval` requires approval of the latest push. The run reads both settings from the branch rules and refuses with `stale_dismissal_off` if either is off. Plan step 2 tests whether GitHub counts a review that the App posts with an older `commit_id` after a push.

Renovate enables GitHub's auto-merge on its pull requests. An approval can trigger a merge within seconds, and the third read accepts a merged pull request if its head and base match the pre-post snapshot.

Before the final pre-post read, record the approver App's review IDs and use `already_approved` only for a currently effective approval at H. After a timeout or another ambiguous POST result, reconcile against those IDs before any retry. Run the third read for a confirmed new approval at H, repeat the pre-post checks before retrying a confirmed failure to write, or stop if the result remains uncertain.

Record the pre-post head and base snapshot in the App's review body. If the job times out, the next run compares the App's existing approval against that snapshot and applies the third-read dismissal rules before it can return `already_approved` or post again.

## The author check

The decision module holds three constants for the self-hosted Renovate App, recorded as [D12](decisions.md): the App's ID, its bot user's ID, and its bot login, `<app-slug>[bot]`. The module fills them from public reads of `GET /users/{login}` and the App's public page when it's built.

The run requires all of these:

- The pull request's `user.id` equals the bot user ID, `user.login` equals the bot login, and `user.type` is `Bot`. Otherwise `author_not_renovate`.
- `GET /repos/{owner}/{repo}/issues/{number}` shows `performed_via_github_app.id` equal to the Renovate App's ID, when GitHub fills that field. Otherwise `author_app_mismatch`. The spike confirms whether GitHub fills it for an App's pull requests, and the check drops out if it doesn't.
- `head.repo.full_name` equals the base repository. Otherwise `fork_head`.
- `GET /repos/{owner}/{repo}/pulls/{number}/commits` lists at most `max_commits` commits, 5 by default. Each has `author.id` equal to the bot user ID, `committer.login` equal to `web-flow`, and `verification.verified` true. Otherwise `too_many_commits`, `commit_not_renovate`, or `signature_unverified`.

A login alone isn't enough, for four reasons:

- A login is a display string. A check that compares it loosely, such as by prefix, matches another App named `renovate-something[bot]`. The public hosted Renovate App's bot is also a different identity from the self-hosted one.
- A user ID never changes, and GitHub never reuses one. Renaming an App changes its slug, which would silently break or redirect a login-only check.
- Anyone can open a pull request from a fork, give its branch a `renovate/` name, and copy Renovate's title. The pull request's author and head repository come from GitHub, and the run checks both.
- GitHub attributes a commit to whichever account owns its author email, and anyone can set that email. A commit that names the bot as author but wasn't created through the API carries no web-flow signature, so the committer and signature checks refuse it. A person who edits the branch in the web interface gets a web-flow signature, but the author is that person, so the author check refuses it.

Renovate's presets set `platformCommit: "enabled"`, which creates commits through the API, signed by GitHub as the App ([appendix A](A-evidence.md)). The exact author and committer fields of those commits are unverified, and the spike records them.

## The required checks

Read the required checks from `GET /repos/{owner}/{repo}/rules/branches/{default_branch}`. The response includes active rules from repository and organization rulesets. Whether metadata read suffices is unverified until the spike.

Each `required_status_checks` rule lists checks as a `context` and an optional `integration_id`. Use the `pull_request` rule from the same response to check the staleness settings.

The tbhb-dev repositories use rulesets, so the run doesn't read classic branch protection. GitHub still enforces any classic required check at merge, so a check the run can't see can block a merge but can't let one through.

The caller supplies the expected workflow ID for each required check from its default-branch workflow definition. The matcher supports only repositories whose required checks all come from one CI workflow. After checking that each required check has an integration and workflow binding, it refuses with `multiple_required_workflows` if those bindings name more than one workflow ID.

Fetch the run by the event's run ID with `GET /repos/{owner}/{repo}/actions/runs/{run_id}` and require the same attempt, workflow ID, `pull_request` event type, head repository ID, and H.

Require one association with the target pull request by number and head branch. Its base repository and target branch must match the pull request snapshot, as must the merge base of H and the association's `base.sha`. Refuse if the association is missing or ambiguous or if GitHub reports a newer attempt of that run.

List jobs with `GET /repos/{owner}/{repo}/actions/runs/{run_id}/attempts/{attempt}/jobs`. For each required check, read the check run by ID from its job's `check_run_url` and require exactly one match for the configured name and integration ID in the workflow run's check suite. The lookup uses IDs, and the result's `head_sha` must equal the `head_sha` of the suite that the run's `check_suite_id` identifies.

The suite's head can equal H or the test merge commit. For a test merge commit, verify that its parents are the CI snapshot's `base.sha` and H. A lookup by H alone could miss a merge-commit check or select a different suite for the same commit.

The spike verifies GitHub's check attribution across reruns and on test merge commits.

A check without evidence from that run and attempt ends the decision with `check_missing`. The workflow rejects reruns on other branches and refuses when the completion lacks any required check. It reads public Actions run and job metadata anonymously and refuses if the evidence is unavailable.

| State in the bound CI snapshot | Reason code | Why |
| --- | --- | --- |
| `completed` with `success` | passes | |
| `queued`, `in_progress`, `waiting`, `requested`, or `pending` | `checks_pending` | A later CI completion starts a fresh decision |
| `completed` with `skipped` | `check_skipped` | A skip supplies no successful CI result. Whether GitHub counts it toward a required check is unverified |
| `completed` with `neutral` | `check_neutral` | The decision requires `success`. Whether GitHub counts `neutral` toward a required check is unverified |
| `failure`, `cancelled`, `timed_out`, `action_required`, `stale`, or `startup_failure` | `check_failed` | |
| No unique check of that name from that App in the bound snapshot | `check_missing` | |
| A required check with no `integration_id` or expected workflow ID | `required_check_unbound` | The producer or workflow is unknown |
| Required checks bound to more than one CI workflow | `multiple_required_workflows` | One workflow must produce all required checks |
| No required checks at all | `no_required_checks` | There is no CI evidence to approve on |

The run refuses with `self_check_required` when the ruleset requires the approving job. That job runs on the default branch and supplies no CI evidence for the pull request.

The Renovate App has checks write and statuses write ([appendix A](A-evidence.md)). Matching the check run's App ID to the ruleset's `integration_id` prevents a `conclusion` result from Renovate from counting.

## The approver App

These are proposals that the org's administrator confirms, under [D3](decisions.md).

- **Name:** `tbhb-dev-pin-approver`, so its reviews show as `tbhb-dev-pin-approver[bot]`.
- **Permissions:** pull requests write, contents read, checks read, and metadata read, which GitHub adds to every App. No others, and no organization permissions.
- **Webhook:** off. The App receives no events, because the workflow does the work.
- **Where it installs:** only on this account, on selected repositories, starting with tbhb-dev/rules.
- **Secrets.** `PIN_APPROVER_CLIENT_ID` and `PIN_APPROVER_PRIVATE_KEY` live in the `pins-approver` environment of each calling repository. The administrator keeps the source key in the approved password manager.

Contents read covers the commits and compare endpoints, and checks read covers check runs. Pull requests write permits posting the review. The spike checks whether GitHub counts the approval, whether the App can dismiss it under the ruleset, and whether the partial fetch works for H and the merge base. [Spike list](A-evidence.md#open-questions-for-the-spike)

The gather step reads public pin metadata through public endpoints. A first-party source that's private needs the App installed there with contents read.

### Minting the token

The job stops before requesting a token if either secret is empty. The pinned `actions/create-github-app-token` action gets `${{ secrets.PIN_APPROVER_CLIENT_ID }}` as `client-id` and `${{ secrets.PIN_APPROVER_PRIVATE_KEY }}` as `private-key`. The `owner` and `repositories` inputs restrict the token to the calling repository.

Both modes request `permission-contents: read` and `permission-checks: read`. For `permission-pull-requests`, the job requests `read` in `evaluate` mode or `write` in `enforce` mode. Only the decision step receives the token. It expires after one hour, and the action revokes it when the job ends.

### What the administrator sets up

1. Create the App under the organization with the name, permissions, and webhook setting in the previous section, installable only on this account.
2. Generate a private key, store it in the approved password manager, and delete the downloaded file.
3. Record the App's ID and bot user ID in the rules maintainer's tracking issue so the tests and logs can name it.

In each calling repository, including the first, complete these tier 2 setup steps before enabling the caller.

1. Install the App on the repository.
2. Create the GitHub Actions environment `pins-approver`. Under **Deployment branches and tags**, choose **Selected branches and tags** and add one branch rule matching the default branch exactly. Leave tag rules and required reviewers unset.
3. Add the App's client ID as the environment secret `PIN_APPROVER_CLIENT_ID` and its private key as `PIN_APPROVER_PRIVATE_KEY`.

## The `workflow_call` interface

The reusable workflow lives at `.github/workflows/renovate-auto-approval.yml` in tbhb-dev/rules (planned).

| Input | Type | Default | Meaning |
| --- | --- | --- | --- |
| `mode` | string | `evaluate` | `evaluate` logs the decision only. `enforce` also posts the approval |
| `check_workflows` | string | none | One `context: workflow ID` per required check, all naming the same CI workflow |
| `pin_files` | string | the globs in [appendix C](C-matcher.md) | One `kind: glob` per line. Callers may only narrow the file set for each built-in kind |
| `allowed_bumps` | string | `patch minor digest` | Space-separated bump levels |
| `first_party_owners` | string | `tbhb-dev` | Owners whose branch and tag pins need the merged-at-source check |
| `first_party_modules` | string | `tbhb_rules*` | `git_override` modules treated as first-party pins |
| `registry` | string | `https://bcr.bazel.build/` | The only registry allowed in `registryFileHashes` keys |
| `max_files` | number | `50` | More changed files refuses with `too_many_files` |
| `max_commits` | number | `5` | More commits refuses with `too_many_commits` |
| `max_download_mb` | number | `256` | A larger artifact refuses with `digest_unverifiable` |
| `opt_out_label` | string | `no-auto-approval` | A pull request with this label gets `opted_out` |
| `runs-on` | string | `ubuntu-26.04-arm` | The runner label |

The reusable workflow exposes `decision`, `reason`, and `head_sha` as outputs. It declares `PIN_APPROVER_CLIENT_ID` and `PIN_APPROVER_PRIVATE_KEY` under `on.workflow_call.secrets`, both optional at the call boundary, and the caller maps both by name under the calling job's `secrets`.

Store the credentials only in `pins-approver`. The caller's expressions have empty values. The spike checks whether GitHub supplies the matching environment secrets to the approving job after admission.

The reusable workflow sets `permissions: {}` at the workflow and job levels because the job uses the App's token. The job declares `environment: pins-approver` and `timeout-minutes: 10`. Its concurrency group is `renovate-auto-approval-${{ github.repository }}-${{ github.event.workflow_run.head_branch }}` with `cancel-in-progress: false`. Runs for one pull request queue, and GitHub keeps only the newest pending run. The job never cancels a run mid-approval, because a cancel between the post and the third read would leave an approval unchecked.

The job's steps, in order:

1. Fail the job if `job.workflow_repository` or `job.workflow_sha` is empty. Pass those values to `actions/checkout` as `repository` and `ref`, with `persist-credentials: false`, and require the checkout's `HEAD` to equal `job.workflow_sha` before proceeding.
2. Require both secrets and request the App token only after checkout verification succeeds.
3. In both modes run `python3 -I -S "$RULES_CHECKOUT/repo/private/renovate_approval/__main__.py"`, where `RULES_CHECKOUT` points to the verified checkout. The entry point appends only the checkout root, derived from its resolved script path, to `sys.path` before importing the decision package by its absolute name. The flags keep the working directory, `PYTHONPATH`, and site packages out of the import path.
4. Upload `decision.json` as an artifact named `renovate-auto-approval-<pull request number>-<run attempt>`, kept for 90 days.

### The call a repository adds

```yaml
# .github/workflows/renovate-auto-approval-caller.yml
name: Renovate auto-approval

on:
  workflow_run:
    workflows: [ci]
    types: [completed]

permissions: {}

jobs:
  approve:
    if: >-
      github.event.workflow_run.event == 'pull_request' &&
      github.event.workflow_run.head_repository.full_name == github.repository
    permissions: {}
    uses: tbhb-dev/rules/.github/workflows/renovate-auto-approval.yml@<40-hex commit>  # main
    with:
      mode: evaluate
      check_workflows: 'conclusion: <ci workflow ID>'
    secrets:
      PIN_APPROVER_CLIENT_ID: ${{ secrets.PIN_APPROVER_CLIENT_ID }}
      PIN_APPROVER_PRIVATE_KEY: ${{ secrets.PIN_APPROVER_PRIVATE_KEY }}
```

The `if:` is a cheap filter that saves a runner for fork and push runs, not a security check. The module repeats both checks against the API.
