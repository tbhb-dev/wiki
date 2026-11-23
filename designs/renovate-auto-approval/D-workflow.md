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

The caller uses `workflow_run` with `types: [completed]` and lists every workflow that reports a required check. Each completion triggers a decision. [D5](decisions.md)

- To inspect pull request changes, the job reads the diff and file text as data through Git objects and the API. It never checks out or executes pull request code. CI artifacts stay outside the job.
- GitHub runs the caller's workflow definition from the default branch. The job runs code from tbhb-dev/rules at the commit that definition pins.
- In the reusable workflow, the approving job declares `environment: pins-approver`. Before the job starts, GitHub checks the run's ref against the environment policy in the calling repository. Only runs from the default branch can read the environment secrets.
- No pull request field, such as a title or a branch name, reaches a `run:` script through a `${{ }}` expression. The script reads the event payload from `GITHUB_EVENT_PATH` and gets everything else from the API.

zizmor's `dangerous-triggers` audit flags `workflow_run`. Each caller adds an ignore entry for its caller file, with a comment that points here.

## The exact-head binding

GitHub has no compare-and-set for a review: the review endpoint can't refuse a post because the head moved. The binding therefore uses three reads and a bound review.

1. **Read once.** The run takes H from `workflow_run.head_sha` in the event payload, the commit CI ran on. No later step reads H from anywhere else.
2. **Find the pull request.** `GET /repos/{owner}/{repo}/commits/{H}/pulls` must return exactly one open pull request whose head repository is this repository and whose head is H. Otherwise the run ends with `pr_not_unique` or `fork_head`.
3. **First read.** `GET /repos/{owner}/{repo}/pulls/{number}` must show `head.sha` equal to H, or the run ends with `head_mismatch_at_start`.
4. **Gather and decide** against H only. Every API call that takes a ref passes H or the merge base, never a branch name.
5. **Second read, just before approval.** Recheck the required checks, then re-read the pull request. The checks must still pass and the pull request must still show head H, `state` open, `draft` false, and no opt-out label. A changed head ends the run with `head_moved`. In `evaluate` mode the run stops after recording the decision.
6. **Bound post.** In enforce mode, send `POST /repos/{owner}/{repo}/pulls/{number}/reviews` with `event: APPROVE` and `commit_id: H`. The run requires the response's `commit_id` to equal H.
7. **Third read.** The pull request read once more must show head H, or merged with merge base and head unchanged. If the head moved, the run calls `PUT .../reviews/{review_id}/dismissals` and logs `head_moved_after_approval`. A ruleset that restricts dismissals can refuse that call, which the spike checks, and the stale dismissal still applies.

The reads check for head changes, and the response's `commit_id` confirms the review targets H. A push can land between the second read and the post. The workflow relies on two ruleset settings to guard that interval. `dismiss_stale_reviews_on_push` voids stale approvals on push, and `require_last_push_approval` requires approval of the latest push. The run reads both settings from the branch rules and refuses with `stale_dismissal_off` if either is off. Plan step 2 tests whether GitHub counts a review that the App posts with an older `commit_id` after a push.

Renovate enables GitHub's auto-merge on its pull requests. An approval can trigger a merge within seconds, and the third read accepts a merged pull request whose head is still H.

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

The run reads the required checks from `GET /repos/{owner}/{repo}/rules/branches/{default_branch}`. That endpoint returns every active rule for the branch from both repository and organization rulesets. It's expected to need only metadata read, not administration, and the spike confirms which permission it takes. Each `required_status_checks` rule lists checks as a `context` and an optional `integration_id`. The same response carries the `pull_request` rule that the head binding reads.

The tbhb-dev repositories use rulesets, so the run doesn't read classic branch protection. GitHub still enforces any classic required check at merge, so a check the run can't see can block a merge but can't let one through.

For each required check, the run calls `GET /repos/{owner}/{repo}/commits/{H}/check-runs?check_name={context}&app_id={integration_id}&filter=latest`. It decides by the latest run of that name from that App:

| State at H | Reason code | Why |
| --- | --- | --- |
| `completed` with `success` | passes | |
| `queued`, `in_progress`, `waiting`, `requested`, or `pending` | `checks_pending` | That check's completion fires another `workflow_run`, which decides again |
| `completed` with `skipped` | `check_skipped` | GitHub lets a skipped required check pass, but a skip means CI didn't test this head |
| `completed` with `neutral` | `check_neutral` | GitHub counts it as a pass, and the run doesn't |
| `failure`, `cancelled`, `timed_out`, `action_required`, `stale`, or `startup_failure` | `check_failed` | |
| No run of that name from that App | `check_missing` | |
| A required check with no `integration_id` | `required_check_unbound` | Any App with checks or statuses write, Renovate's included, could satisfy it |
| No required checks at all | `no_required_checks` | There is no CI evidence to approve on |

The run excludes the workflow's own check, and no ruleset should ever list it. A `workflow_run` job attaches its check to the default branch's head, not to H, so it never shows at H. If a ruleset named it as required anyway, it would still be running when the run reads it, and the run would wait on itself forever. Its result also says nothing about the change. The run refuses with `self_check_required` when the required list names its own job, so the misconfiguration shows up in the log.

Renovate's App holds checks write and statuses write ([appendix A](A-evidence.md)). Binding each required check to an `integration_id` and passing `app_id` is what stops a check run that Renovate posts named `conclusion` from counting.

## The approver App

These are proposals that the org's administrator confirms, under [D3](decisions.md).

- **Name:** `tbhb-dev-pin-approver`, so its reviews show as `tbhb-dev-pin-approver[bot]`.
- **Permissions:** pull requests write, contents read, checks read, and metadata read, which GitHub adds to every App. No others, and no organization permissions.
- **Webhook:** off. The App receives no events, because the workflow does the work.
- **Where it installs:** only on this account, on selected repositories, starting with tbhb-dev/rules.
- **Secrets.** `PIN_APPROVER_CLIENT_ID` and `PIN_APPROVER_PRIVATE_KEY` live in the `pins-approver` environment of each calling repository. The administrator keeps the source key in the approved password manager.

Contents read covers the commits and compare endpoints and the Git fetch of H and the merge base. Checks read covers the check runs. Pull requests write covers the review and its dismissal. Lookups in third-party public repositories, such as a tag in `actions/checkout`, work with any token. A first-party source that's private needs the App installed there with contents read.

### Minting the token

The approving job requests an installation token with `actions/create-github-app-token` at a pinned commit. Once GitHub admits the job to `pins-approver`, the action gets `${{ secrets.PIN_APPROVER_CLIENT_ID }}` as `client-id` and `${{ secrets.PIN_APPROVER_PRIVATE_KEY }}` as `private-key`. The token covers only the calling repository, as specified by `owner` and `repositories`.

Both modes request `permission-contents: read` and `permission-checks: read`. For `permission-pull-requests`, the job requests `read` in `evaluate` mode or `write` in `enforce` mode. Only the decision step receives the token. It expires after one hour, and the action revokes it when the job ends.

### What the administrator sets up

1. Create the App under the organization with the name, permissions, and webhook setting in the previous section, installable only on this account.
2. Generate a private key, store it in the approved password manager, and delete the downloaded file.
3. Record the App's ID and bot user ID in the rules maintainer's tracking issue so the tests and logs can name it.

In each calling repository, including the first, complete these steps before enabling the caller.

1. Install the App on the repository.
2. Create the GitHub Actions environment `pins-approver`. Under **Deployment branches and tags**, choose **Selected branches and tags** and add one branch rule matching the default branch exactly. Leave tag rules and required reviewers unset.
3. Add the App's client ID as the environment secret `PIN_APPROVER_CLIENT_ID` and its private key as `PIN_APPROVER_PRIVATE_KEY`.

## The `workflow_call` interface

The reusable workflow lives at `.github/workflows/renovate-auto-approval.yml` in tbhb-dev/rules (planned).

| Input | Type | Default | Meaning |
| --- | --- | --- | --- |
| `mode` | string | `evaluate` | `evaluate` logs the decision only. `enforce` also posts the approval |
| `pin_files` | string | the globs in [appendix C](C-matcher.md) | One `kind: glob` per line. The caller can narrow or add files for a known kind, never add a kind |
| `allowed_bumps` | string | `patch minor digest` | Space-separated bump levels |
| `first_party_owners` | string | `tbhb-dev` | Owners whose `main` pins need the merged-at-source check |
| `first_party_modules` | string | `tbhb_rules*` | `git_override` modules treated as first-party pins |
| `registry` | string | `https://bcr.bazel.build/` | The only registry allowed in `registryFileHashes` keys |
| `max_files` | number | `50` | More changed files refuses with `too_many_files` |
| `max_commits` | number | `5` | More commits refuses with `too_many_commits` |
| `max_download_mb` | number | `256` | A larger artifact refuses with `digest_unverifiable` |
| `opt_out_label` | string | `no-auto-approval` | A pull request with this label gets `opted_out` |
| `runs-on` | string | `ubuntu-26.04-arm` | The runner label |

The reusable workflow exposes `decision`, `reason`, and `head_sha` as outputs. The approving job reads both credentials directly from `pins-approver` through its `secrets` context.

The reusable workflow sets `permissions: {}` at the workflow and job levels because the job uses the App's token. The job declares `environment: pins-approver` and `timeout-minutes: 10`. Its concurrency group is `renovate-auto-approval-${{ github.repository }}-${{ github.event.workflow_run.head_branch }}` with `cancel-in-progress: false`. Runs for one pull request queue, and GitHub keeps only the newest pending run. The job never cancels a run mid-approval, because a cancel between the post and the third read would leave an approval unchecked.

The job's steps, in order:

1. Check out tbhb-dev/rules at `github.job_workflow_sha`, the commit of the reusable workflow itself, with `persist-credentials: false`. It needs no token, because the repository is public.
2. Mint the App token.
3. Run `python3 -I -m repo.private.renovate_approval` with the token, the inputs, and the event path in its environment. It uses the runner's Python and the standard library, with nothing installed.
4. Upload `decision.json` as an artifact named `renovate-auto-approval-<pull request number>-<run attempt>`, kept for 90 days.

### The call a repository adds

```yaml
# .github/workflows/renovate-auto-approval.yml
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
```

The `if:` is a cheap filter that saves a runner for fork and push runs, not a security check. The module repeats both checks against the API.
