---
type: Design Appendix
title: In-house merge queue enforcement
description: The ways to make the queue the only path to main without GitHub's merge queue rule, and the GitHub features and plan support each relies on.
tags: [merge-queue, rulesets, github, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# In-house merge queue enforcement

Back to [the design](README.md).

## Constraints

Any option must keep what the main rulesets enforce today (proven: live read of ops' branch rules, 2026-11-24):

- no bypass actors (existing: the checked-in ruleset files and the standing rule; the live rules read doesn't return bypass actors)
- required signatures
- merge commits only, through `allowed_merge_methods: ["merge"]`
- one approval, dismissed on push, with last-push approval and resolved threads
- the `conclusion` check from GitHub Actions, pinned by `integration_id` 15368

Changing who can land on main is an operator decision, D2.

## The options

| Option | GitHub feature it needs | Works on Team for private repositories | Keeps no bypass | Verdict |
| --- | --- | --- | --- | --- |
| E1. A required check that only the queue's App can set | The ruleset rule `required_status_checks` with an `integration_id` on the check | Yes (proven: ops is private and runs this rule today with Actions' ID) | Yes | Recommended |
| E2. Only the queue's App may push to main, through a ruleset | The rule types `update` or `creation` restrict pushes to bypass actors | Rulesets yes (proven). The restriction works by naming the App as a bypass actor | No | Rejected, conflicts with no bypass |
| E3. Only the queue's App may push to main, through classic branch protection | The classic protection setting that restricts who can push to matching branches, listing users, teams, or Apps | Yes, per GitHub's docs: branch restrictions cover every repository of a Team organization (unverified live) | Arguably, since it isn't a ruleset bypass, but it grants the App sole push rights | Fallback only |
| E4. Convention and detection | None. `alpha merge` enqueues, and the queue flags any push to main it didn't make | Yes | Yes | The state before D2, and the fallback if D2 is no |

### E1 in detail

The main ruleset's `required_status_checks` gains a second entry, `{"context": "merge-queue", "integration_id": <queue App ID>}`. GitHub's docs let a required check name an App as its expected source, and then a same-named check from any other App or from Actions doesn't count (per docs, 2026-11-24).

- **Bootstrap.** GitHub offers an App as a check's source only after that App has reported the check in the repository, and the App needs `statuses: write` for that (the reviewer cites GitHub's ruleset docs). The App therefore reports `merge-queue` once on a test pull request before the ruleset names it. Plan step 3 then shows that a same-named check from Actions leaves a test pull request blocked.
- **Head and test merge commit.** GitHub can evaluate required checks on the test merge commit rather than the head when that commit has a status. Plan step 3 runs a pull request carrying both the Actions `conclusion` and the App's check to confirm which commit GitHub reads (per GitHub's troubleshooting docs).
- **Who sets it.** The queue creates the check run on each queued head when it admits the entry, as `in_progress`. A removal or an abort sets it to `failure` or `action_required` with the reason. The queue's admission check ignores its own `merge-queue` check, so an earlier `action_required` can't block a later admission.
- **One success at a time.** In each repository, at most one head carries `success`. The queue sets it on the next entry only after the entry ahead has merged and main's new tree equals that entry's candidate tree. Before it sets `success`, it checks that `git merge-tree` of main's tip and the head gives the entry's candidate tree.
- **Why order holds.** A later entry was tested on top of the entries ahead of it, so it mustn't merge first. Since no later entry has `success` until those entries have merged, a person, Renovate, or GitHub's auto-merge can't merge out of order. Anyone who merges the one green head lands the exact tested combination, and the queue treats an already-merged head as done.
- **A stale success.** If the queue crashes after setting `success`, a later direct merge could land on a main that moved. Recovery flips every unmerged `success` back before the queue does anything else ([appendix F](F-architecture.md#failure-and-recovery)). The remaining window lasts from the crash until the group resumes. It's security debt.
- **Renovate.** Renovate's `platformAutomerge` asks GitHub to merge once required checks pass. With E1 that waits for the queue, so Renovate configs in queue repositories add the `merge-queue` label, and the queue admits Renovate pull requests under a known-bot rule (proposed in the earlier design, not built).
- **The operator.** With no bypass, the operator merges by hand only after taking the check out of the ruleset ([appendix F](F-architecture.md#the-operators-stop)). That's the same as today for every other required check.

### Why E2 and E3 lose

Both block every push except the App's, including GitHub's own merge on behalf of anyone else. That's stronger than E1, but a down queue blocks every merge, and both grant the App sole write rights to main, an authority change. E2 also needs a bypass actor on a main ruleset, which the standing rule forbids.

## Landing: merge API or the tested commit

GitHub's queue fast-forwards main to the exact commit it tested. Copying that needs a push to main, so it falls under E2 or E3.

| Way to land | Same commit SHA as tested | Signed | Needs | Choice |
| --- | --- | --- | --- | --- |
| The merge endpoint, `PUT /repos/{owner}/{repo}/pulls/{number}/merge` with `merge_method: merge` and `sha` set to the head | No, the same tree when main hasn't moved | GitHub signs merges it creates (existing: ops requires signatures and lands merge commits through `gh pr merge`; the docs leave the merge endpoint's signing unsettled, so plan step 1 checks a real commit) | Nothing new | Chosen, D11 |
| Push the candidate commits to main | Yes | Only if GitHub signs them. Its docs verify an App's API commits without custom author, committer, or signature details, and leave the Git database API unsettled | A bypass actor or E3 | Not chosen |

The merge endpoint passes through every ruleset check at merge time, which makes it a second line of admission. After each merge, the queue compares main's new tree with that entry's candidate tree. A mismatch can only come from a merge GitHub resolved differently than `git merge-tree`, or a push the queue didn't make. It pauses that repository and raises an alert, since main may hold an untested combination.

## Branch rulesets that touch candidates

- **The branch-prefix ruleset** restricts branch creation outside `agent/**` and `renovate/**`, with only the operator as bypass actor (existing: rules' checked-in file, enforcement disabled there; the live state is unverified). Its exclude list gains `refs/heads/merge-queue/**/*` before candidates start, so the App needs no bypass.
- **The agent trailer ruleset** targets `agent/**` only (existing: rules' checked-in file), so candidate branches don't meet it.
- **A queue-branch ruleset** targets `refs/heads/merge-queue/**/*`, with the queue App as its one bypass actor (D3). It goes on in plan step 1 and is verified first. Then the queue checks the prefix for earlier branches and runs, switching prefix and repeating if it finds any, all before the first candidate push (D16). Candidates use the pull request cache token until plan step 5.

## CI on candidate pushes

Candidates arrive as `push` events, so each workflow has to tell them apart from pushes to main (existing: tools `ci.yml` keys the cache environment, the wheel job, and the release job on `github.event_name == 'push'`):

- Release and deploy jobs key on `github.ref == 'refs/heads/main'`.
- The cache environment keys on main, and on `merge-queue/main/*` from plan step 5 (D3). Rollback removes that branch policy before it touches the ruleset.
- The `on.push.branches` list gains `merge-queue/**`, which in workflow filters does cross slashes.

tbhb-dev/rules' reusable workflows and the repository baseline carry this change once, and each repository picks it up through its pin (new). GitHub's docs say a push with an App installation token starts workflows, unlike one with `GITHUB_TOKEN`. Plan step 1 confirms it live before anything else depends on it.
