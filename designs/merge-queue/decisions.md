---
type: Design Appendix
title: In-house merge queue decisions
description: The open and decided questions of the in-house merge queue design.
tags: [merge-queue, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words ALLGREEN retarget fnmatch -->

# In-house merge queue decisions

Back to [the design](README.md).

## Open

None.

## Decided

### D17. The D1 App also gets `workflows: write`

- **Decided:** 2026-11-25, by the operator, as an amendment to D1.
- **Rule:** the admin VM role adds `workflows: write` when it creates the App. Only the queue's own candidate pushes use it. Plan step 1 pushes a candidate that changes a workflow file with exactly the D1 permissions.
- **Why:** two pull requests can change different parts of one file under `.github/workflows`, and their candidate merge then holds content that exists on no branch. GitHub refuses an App's push that changes workflow files unless it has `workflows: write` (per docs, as the reviewer cites them), so a valid group would stall before CI starts.
- **Rejected:** testing each entry that changes `.github/workflows` alone from its own head, which must already contain main's tip and needs a fresh approval whenever main moves.
- **Undo:** remove the permission. Candidates that change workflow files then fail to push, and the queue removes those entries with a reason.
- **Record:** the operator's decision of 2026-11-25.

### D3. The queue's branches are queue-only and get the main cache token

- **Decided:** 2026-11-25, by the operator. The review of this design then fixed the branch patterns and the order below, which keep the answer's intent.
- **Rule:** a branch ruleset targeting `refs/heads/merge-queue/**/*` restricts creation, updates, and deletion, and names the D1 App as its only bypass actor. Under GitHub's `fnmatch` patterns a trailing `**` doesn't cross the slash in `merge-queue/main/<entry>`, so the ruleset needs `**/*` (per the reviewer's reading of the docs; plan step 1 tests it on the real path). The `cache` environment's branch policy allows `merge-queue/main/*`, so candidates read and write the main cache namespace and warm main's next run and later candidates. Plan step 5 adds the environment policy and measures what candidates gain. Its timings are a plan step, not a condition on this decision.
- **Why:** this loosens a trust boundary twice. It adds a bypass actor, on a branch ruleset outside main, and it gives the main cache token to branches beyond main. Without the ruleset, any repository writer could push to `merge-queue/main/` and get that token. GitHub's docs say rulesets with App bypass actors and environment branch policies both work on Team private repositories (per docs, 2026-11-24).
- **Undo:** remove `merge-queue/main/*` from the environment's policy first, then delete the ruleset. The reverse order would leave a window where a writer's branch gets the token.
- **Record:** the operator's decision of 2026-11-25.

### D2. In each repository where the queue is live, only the queue lands changes on main

- **Decided:** 2026-11-25, by the operator.
- **Rule:** the main ruleset requires a `merge-queue` check pinned to the D1 App, added per repository when the queue goes live there. `tbhbdev alpha merge` enqueues instead of merging in those repositories, and the operator keeps the manual path in [appendix F](F-architecture.md#the-operators-stop).
- **Why:** agents already merge only through `tbhbdev alpha merge` (D10), but people and the merge command itself could still land a combination nobody tested, and then the queue couldn't keep main green.
- **Undo:** remove the check from the ruleset.
- **Record:** the operator's decision of 2026-11-25.

### D1. tbhb-dev creates a dedicated GitHub App for the merge queue

- **Decided:** 2026-11-25, by the operator.
- **Rule:** the admin VM role creates the App on the org at no cost, with contents, pull requests, checks, and statuses set to write, workflows set to write (D17), actions and metadata set to read, and installs it on all repositories. Its key stays in the tbhb-dev 1Password account, read through Connect. GitHub lists an App as a check's expected source only after it reports that check, which needs statuses write. The App gets no issues write: flake reports go to the dashboard.
- **Why:** the required check in D2 means something only if no agent's App can create it. Reusing the pull-requests App would let any agent run forge the queue's check.
- **Undo:** suspend or delete the App, and rotate its key.
- **Record:** the operator's decision of 2026-11-25.

### D16. The queue's branches are protected before their first use, and the main cache token waits for step 5

- **Decided:** 2026-11-24, by the merge queue maintainer role. Revised 2026-11-25 after review.
- **Rule:** protecting a namespace that writers could already use doesn't settle who created what is in it: an old branch can still dispatch a workflow, and a rerun keeps its original ref. So plan step 1 runs in this order in each repository. First, the D3 ruleset goes on, and a writer's test push to `merge-queue/main/<entry>` must be refused. Second, with the ruleset verified, the queue lists branches and workflow runs under `merge-queue/`. Third, if any exist, the queue picks a fresh prefix, the design's patterns follow it, and the steps repeat on the new prefix. Only then does the App make its first candidate push. Candidates use the pull request cache token until plan step 5 adds the environment policy.
- **Undo:** remove any candidate branch policy from the `cache` environment first, as D3's undo does, then delete the ruleset. Candidates still bind every result to a commit SHA, so the queue keeps working without the ruleset.
- **Record:** this design's pull request.

### D15. GitHub's native queue stays on the public repositories until step 6

- **Decided:** 2026-11-24, by the merge queue maintainer role.
- **Rule:** tools, rules, and event-gateway keep GitHub's merge queue until the in-house queue runs live on ops and agent-config.
- **Undo:** move them earlier or later. Each move is a ruleset change per repository.
- **Record:** this design's pull request.

### D14. Starting queue settings

- **Decided:** 2026-11-24, by the merge queue maintainer role.
- **Rule:** the first live release runs serial: one entry per group, one candidate building at a time. Later defaults are groups of 1 to 5, every entry tested on its own candidate as `ALLGREEN` does, two candidates building at once, a fill wait of 0, and a 60-minute CI timeout, set per repository in the worker's config.
- **Undo:** change the config. A change takes effect for groups built after it.
- **Record:** this design's pull request.

### D13. Labels drive the queue

- **Decided:** 2026-11-24, by the merge queue maintainer role.
- **Rule:** the `merge-queue` label enqueues. A `hold` label keeps an entry out of groups, and an `urgent` label puts an entry first when the operator's account set it. Both labels come from the earlier design's holds table, which proposed them and nothing has built yet.
- **Undo:** replace the labels with signals sent by a command. Nothing outside the queue and `alpha merge` reads them.
- **Record:** this design's pull request.

### D12. A Go worker in a new public repository

- **Decided:** 2026-11-24, by the merge queue maintainer role.
- **Rule:** the worker and workflows live in tbhb-dev/merge-queue under Apache-2.0, in Go like the event gateway. History that outlives a workflow sits in one SQLite file until the shared Postgres server exists. Commit messages come from `tbhbdev alpha merge`, so the message rules keep one home.
- **Undo:** move the code into another repository, or the history into Postgres. GitHub holds enough to rebuild the history.
- **Record:** this design's pull request.

### D11. Merge through the API, not by pushing the tested commit

- **Decided:** 2026-11-24, by the merge queue maintainer role.
- **Rule:** the queue merges each pull request through GitHub's merge endpoint with a merge commit, then checks that main's tree equals the tested tree. It never pushes to main, which keeps the main rulesets free of bypass actors.
- **Undo:** switch to pushing the tested commits, which needs a bypass actor and so an operator decision.
- **Record:** this design's pull request.

### D10. Agents merge only through `tbhbdev alpha merge`

- **Decided:** 2026-11-23, by the operator.
- **Rule:** the guard refuses every other merge path for agents, including direct API merges. The queue's worker isn't an agent run, so its App merges through the API, and `alpha merge` becomes the agents' way to enqueue.
- **Undo:** an operator decision.
- **Record:** the operator's decision of 2026-11-23.

### D9. Temporal orchestrates the merge queue

- **Decided:** 2026-11-22, by the operator.
- **Rule:** the review and merge queue runs as Temporal workflows, self-hosted on hosts tbhb-dev owns, with the event gateway in front. Older queue designs are inspiration, not references.
- **Undo:** an operator decision. The record's own undo returns the review queue to a model-free worker.
- **Record:** the operator's decision of 2026-11-22.

### D8. Merges never wait on a time delay

- **Decided:** 2026-11-10, by the operator.
- **Rule:** the queue's fill wait defaults to 0 minutes. A group forms from whatever is queued when a build slot opens.
- **Undo:** an operator decision.
- **Record:** the operator's decision of 2026-11-10, 16:45Z, as cited in the earlier design's decided list.

### D7. The worker runs as a Unix service account on a service host

- **Decided:** 2026-11-12, by the operator.
- **Rule:** the worker is a Unix account without an agent harness or sudo, on a service host, never an agent Mac, and the build adds no GitHub user account. The earlier record routes its GitHub writes through the existing Apps' tokens, and the new App this design needs is D1.
- **Undo:** an operator decision.
- **Record:** the operator's decision of 2026-11-12, as the earlier design's D2 cites it.

### D6. A reconcile read every 15 minutes

- **Decided:** 2026-11-12, by the operator.
- **Rule:** the worker may read GitHub every 15 minutes to catch lost events, as a sanctioned exception to the no-polling rule.
- **Undo:** an operator decision.
- **Record:** the operator's decision of 2026-11-12, as the earlier design's D1 cites it.

### D5. The merge commit carries a `Merged-by` trailer

- **Decided:** 2026-11-12, by the operator.
- **Rule:** the commit that lands a pull request names its merger in `Merged-by`. The queue uses the tool form, with tool name, version, and instance hash, and no host names.
- **Undo:** an operator decision.
- **Record:** [`adrs/0014-merge-commits-record-the-merger.md`](../../adrs/0014-merge-commits-record-the-merger.md), and the earlier design's D4.

### D4. Pull requests land through merge commits only

- **Decided:** 2026-11-22, by the operator.
- **Rule:** repositories allow only merge commits, never squash or rebase, so stacked pull requests keep their parents' commit IDs.
- **Undo:** an operator decision.
- **Record:** [`adrs/0020-pull-requests-land-through-merge-commits.md`](../../adrs/0020-pull-requests-land-through-merge-commits.md), decided 2026-11-22.
