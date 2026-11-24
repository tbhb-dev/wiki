---
type: Design Draft
title: Renovate auto-approval for pin-only updates
description: A reusable workflow and a dedicated GitHub App that approve Renovate pull requests that only move pins, for the rules maintainer and every repository that calls it.
tags: [renovate, approvals, github-actions, design]
status: draft
owner: Rules maintainer
updated: "2026-11-23"
decisions_open: 3
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Renovate auto-approval for pin-only updates

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Rules maintainer | 2026-11-23 | 3 open decisions in [decisions.md](decisions.md), about 10 minutes |

## Summary

In repositories that require an approval, Renovate pull requests that only move a pin still wait for a person. CI covers the pins its checks exercise. This repository's CI doesn't run mise or use `mise.lock` for validation.

This design adds a reusable workflow in tbhb-dev/rules for repositories where one CI workflow reports every required check. A dedicated GitHub App approves a pull request only when the Renovate App authored it, the diff moves only recognized pins and their companion entries, and the required checks passed in that pull request's CI run. The workflow checks the head and base again before posting.

For the operator, pin-only updates stop needing a hand approval, and every decision leaves a reason code in the job summary.

## Proposal

Once built, the design has four pieces:

- **The decision module**, `repo/private/renovate_approval/` in tbhb-dev/rules (planned). A pure function, `decide(diff, metadata, checks)`, returns `(approve, reason)` with no network access. A gather step around it reads GitHub and hands it plain data.
- **The reusable workflow**, `.github/workflows/renovate-auto-approval.yml` in tbhb-dev/rules (planned). Its approving job never checks out or executes pull request code.
- **The approver App**, a GitHub App that the organization's administrator creates and installs on the repositories that opt in (planned, [D3](decisions.md)). It can write pull request reviews and read contents and checks.
- **A caller workflow** in each repository that runs from the default branch's workflow definition after CI completes.

The rules maintainer maintains the module, the reusable workflow, and its tests. Each repository's maintainer decides when to switch the caller from `evaluate` to `enforce`. The job writes its decision to the job summary and a run artifact.

The administrator configures the tier 2 `pins-approver` environment in each calling repository for the default branch ref only, with required reviewers unset. The reusable workflow declares both App secrets under `on.workflow_call.secrets`, and the caller maps them by name. The credentials live only in the environment, and the spike verifies that the approving job receives them there. [Workflow interface](D-workflow.md#the-workflow_call-interface)

The operator's direction of 2026-11-23 sets the scope ([D1](decisions.md)). This design adds the mechanics, the refusal list, and the rollout.

## How it works

### One run, step by step

1. **Trigger.** A `workflow_run` event fires when a listed CI workflow completes for a pull request. The workflow takes H, the head of the pull request, from that event. It also records the run ID, attempt, and head repository.
2. **Find the pull request.** The run lists the open pull requests whose head is H and requires exactly one, from a branch in the same repository.
3. **Bind the pull request.** The run requires head H in the calling repository and a target of its default branch. It records the base repository, target branch, and merge base.
4. **Gather the evidence.** The run reads the author, the commits and their signatures, the required check results from the triggering CI snapshot, and the zero-context diff from the merge base to H. It also reads the facts each pin needs, such as which commit a tag names.
5. **Decide.** `decide()` applies every rule to that data and returns one decision and one reason code, plus the full list of findings.
6. **Re-read and approve.** The run rechecks the required checks, then confirms that H, the base repository, target branch, and merge base still match. The pull request must still be open, with no draft status or opt-out label. In `enforce` mode the run posts the review with `commit_id` set to H.
7. **Check after.** The run reads the pull request a third time. If the head or base changed, it attempts to dismiss its review and logs the result.
8. **Log.** Every run, approve or not, writes the decision, the reason code, H, and the findings to the job summary and to a JSON run artifact.

[Appendix D](D-workflow.md) gives the trigger comparison, the head binding, and the GitHub API calls.

### What earns an approval

All of these hold at H, or the run refuses:

- **Author.** The pull request's author has the Renovate App's bot user ID, login, and type `Bot`. Every commit has that author, GitHub's web-flow committer, and a verified signature. A login match alone fails, as [appendix D](D-workflow.md#the-author-check) explains.
- **Shape.** The branch lives in the base repository, the pull request targets the default branch, and it's open and not a draft.
- **Pin-only diff.** Each changed line replaces a recognized pin value or belongs to a companion lockfile change the matcher allows. Text pin lines change only in their value slots. [Appendix C](C-matcher.md) gives the grammar and exemptions, including the exclusion for the reusable approver workflow file.
- **Pin facts.** Each new pin is an upgrade within the allowed bump levels. A tag names exactly the new commit, the source's default branch contains a branch pin's commit, and a new digest matches the downloaded artifact.
- **Checks.** Every required check concluded `success` for the expected GitHub App in the triggering CI run and attempt for this pull request. Pending, skipped, neutral, missing, and failed checks refuse. Required checks bound to more than one workflow refuse with `multiple_required_workflows`.
- **Ruleset.** The branch's rules require an approval. The run refuses with `stale_dismissal_off` if either stale approval dismissal on push or last-push approval is off.

### Pin files it recognizes

| File | Pin lines it accepts |
| --- | --- |
| Workflow and action YAML files, excluding the file that defines the reusable approver | `uses: owner/repo[/path]@<40 hex>  # <version or main>` |
| `MODULE.bazel` and listed `.bzl` files | `bazel_dep` version, the shared-rules `git_override` commit, and the URL version with its digest in `http_archive`, `http_file`, and `archive_override` |
| `mise.toml` | A version in `[tools]` |
| `mise.lock`, `MODULE.bazel.lock` | Companion entries for a pin the same pull request moves, checked as parsed documents |

The reusable workflow exposes a [`pin_files` input](D-workflow.md#the-workflow_call-interface). The preset pins in `.github/renovate.json5` stay out until [D4](decisions.md) says otherwise.

### Modes

The caller sets `mode`. In `evaluate`, the run does everything except post the review, and logs what it would have done. In `enforce`, it posts approvals. A repository starts in `evaluate`, and [appendix F](F-rollout.md) gives the bar for moving to `enforce`.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| GitHub counts an approval whose `commit_id` is an older commit after a push lands between the re-read and the post | An approval could count for a head nobody checked | Rulesets dismiss stale approvals on push and require last-push approval. The post-check dismisses the review. A spike in plan step 2 tests the race | unverified |
| GitHub counts an approval after the target branch changes between the re-read and the post | An approval could count for a different target | The third read detects the changed target and attempts dismissal. Plan step 2 tests stale-review behavior during the race | unverified |
| The App's approval doesn't count toward the required review with contents read only | Enforce mode approves and nothing merges | Plan step 2 tests it. A wider grant goes back to the operator, and nobody adds it alone | unverified |
| A workflow edit runs from a pull request branch | The branch run tries to read the App's key | Plan step 2 tests whether the tier 2 `pins-approver` policy rejects the run ref before the job can read its secrets | unverified |
| A matcher bug accepts a non-pin change | A change that nobody reviewed merges | Fail-closed grammar, table tests for every reason code, an evaluate period, and two tier 2 reviews | new |
| The Renovate App can write checks and statuses | A forged `conclusion` result | The run matches each check's App ID and requires evidence from the triggering CI snapshot | existing |
| A rules change weakens the matcher, then reaches consumers through an auto-approved rules pin | Weaker checks everywhere | Changes to the module are tier 2 in tbhb-dev/rules, the review that the pin inherits | new |
| The zero-context diff comes from a partial fetch the runner can't complete | No decision | The run refuses with `diff_unavailable`, and the next CI completion retries | unverified |
| A busy org exhausts the App's API rate limit | Runs refuse | Each run makes about 30 to 80 calls and refuses with `rate_limited`. The next event retries | new |

[Appendix E](E-failures-and-tests.md) lists every failure mode and its reason code.

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | A decision record that amends the review rules for pin-only Renovate pull requests that [D1](decisions.md) and [D2](decisions.md) call for | The record merges in `adrs/` after its own two reviews | Rules maintainer |
| 2 | The administrator creates the App and sets up `pins-approver` in a scratch repository per [D3](decisions.md). The spike tests that the deployment policy blocks a non-default branch run from reading the secrets. It also checks the other behaviors in [appendix A](A-evidence.md#open-questions-for-the-spike) | The spike's pull request records each result as proven or refuted in [appendix A](A-evidence.md) | Org administrator, then rules maintainer |
| 3 | `decide()` with its table tests, in tbhb-dev/rules | `bazelisk test //repo/private/renovate_approval:all` passes with one case per reason code | Rules maintainer |
| 4 | The gather step, the reusable workflow, and its zizmor and actionlint entries | `bazelisk test //...` passes in tbhb-dev/rules, and a manual run against a closed Renovate pull request logs a decision | Rules maintainer |
| 5 | The `pins-approver` environment and the caller in tbhb-dev/rules, in `evaluate` mode | Every Renovate pull request there for 14 days has a decision artifact | Org administrator and rules maintainer |
| 6 | The decision-log review, then `enforce` in tbhb-dev/rules | The review's PR shows zero wrong approvals and explains every refusal, per [appendix F](F-rollout.md) | Rules maintainer, second reviewer |
| 7 | The `pins-approver` environment and caller in each new repository, starting in `evaluate` | Each repository passes step 6's bar before it enforces | Org administrator and each repository's maintainer |

## Appendices

- [Decisions](decisions.md): 3 open, then the decided record.
- [A. Evidence](A-evidence.md): what the repositories, rulesets, Renovate presets, and GitHub docs show, labeled proven, existing, new, or unverified.
- [B. Alternatives considered](B-alternatives.md): other triggers, other approvers, and dropping the approval requirement.
- [C. The pin-file matcher](C-matcher.md): the grammar for each file kind, the refusals, and the accept and refuse table.
- [D. The workflow](D-workflow.md): the trigger and head binding, the author check, required checks, the App, and the `workflow_call` interface.
- [E. Failure modes and tests](E-failures-and-tests.md): every failure mode, the reason codes, and the test plan for `decide()`.
- [F. Rollout](F-rollout.md): evaluate mode, the first repository, the decision-log review, and what the two tier 2 reviews check.
