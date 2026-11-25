---
type: Design Appendix
title: In-house merge queue GitHub parity and additions
description: Each GitHub merge queue behavior the in-house queue copies, and each behavior it adds.
tags: [merge-queue, github, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words ALLGREEN HEADGREEN retarget retargets -->

# In-house merge queue GitHub parity and additions

Back to [the design](README.md).

## What GitHub's queue does well

The queue copies each of these. GitHub's names appear in code font where the setting has one, so a reader can map today's ruleset values onto the new config.

| GitHub behavior | How the in-house queue does it | Evidence |
| --- | --- | --- |
| Each entry is tested on top of the projected main, meaning main plus every entry ahead of it | Each entry's candidate is the chain of merge commits on main's tip up to and including that entry, pushed to its own branch | new, copying GitHub's model (unverified: docs not re-read in this pass) |
| Required checks run on the `merge_group` event | Required checks run on a `push` to `merge-queue/main/<entry>`. Private repositories never get `merge_group` events, so a candidate push stands in | new |
| `ALLGREEN` grouping, the API's name for the setting "Only merge non-failing pull requests": every pull request in a group must pass its required checks | The only mode. Each entry gets its own candidate and CI run, and the queue merges entries one at a time, each only after its own candidate passed | existing: tools and rules use `ALLGREEN` today; the meaning is per GitHub's docs, which describe the setting without the API name |
| `HEADGREEN` grouping, the setting turned off: a failed pull request can join a group if the last one passes | Not copied. GitHub lands a whole group by moving main once. The queue lands through the merge endpoint one pull request at a time, so after a refused second merge, main would hold a first entry nobody tested alone | per GitHub's docs |
| `min_entries_to_merge`, `max_entries_to_merge`, `min_entries_to_merge_wait_minutes` | Group size 1 to N, with a fill wait of 0 by default under D8 | existing: tools uses 1, 5, and 5 minutes |
| `max_entries_to_build`: how many candidates build at once | Build concurrency per repository, plus one ceiling across all repositories so runners aren't swamped | existing: tools and rules use 1 |
| `check_response_timeout_minutes` | A CI timeout per repository. A timed-out group counts as an infrastructure failure, retried once | existing: tools uses 60 |
| A failing entry leaves the queue, and every candidate behind it rebuilds without it | The same. Each entry's own candidate names the failing entry | new |
| A push to a queued pull request removes it from the queue | A `synchronize` event removes the entry, since the approval no longer covers the head | new |
| Read-only temporary branches, `gh-readonly-queue/main/...` | `merge-queue/main/<entry>` branches, deleted after each group. A ruleset makes them queue-only before their first use (D3, D16), and each result also binds to a commit SHA | new |
| Status on the pull request: queued, position, and why it left | The `merge-queue` check run shows position, group, the job and step now running, and each wait reason. One comment explains a removal | new |
| The merge method comes from the queue's settings | Merge commits only, under D4 | existing: [ADR 0020](../../adrs/0020-pull-requests-land-through-merge-commits.md) |

## What GitHub's queue lacks

| Addition | What it does | Plan step |
| --- | --- | --- |
| Private repositories on the Team plan | The whole point. Nothing in the queue depends on GitHub's merge queue feature | 2 and 3 |
| One queue for every repository | One worker, one dashboard, and one status command across all repositories, with a shared ceiling on concurrent builds | 2 |
| Stacked pull requests | A child whose base is a queued parent waits behind it. When the parent merges, the queue retargets the child to main and keeps its place. Whether the retarget dismisses the child's approval is unverified | 4 |
| Dependencies across repositories | A `Depends-on: owner/repo#N` line in the body holds the entry until that pull request merges. There's no atomic merge across repositories, and the queue doesn't pretend otherwise | after 6 |
| Priority | The operator's `urgent` label puts an entry at the front. Groups already building behind its new position rebuild | 4 |
| Proven-tree skip | A candidate whose tree already passed CI skips CI ([appendix D](D-checks-and-flakes.md#proven-tree-skip)) | 5 |
| Flake handling | One automatic retry for infrastructure failures and for tests marked flaky, with every retry recorded ([appendix D](D-checks-and-flakes.md#flakes)) | 5 |
| Timing per job and step | Mimir metrics and Loki logs for each candidate's jobs and steps, so a 1400-second pass shows where the time went | 5 |
| Clear answers for a stuck agent | `tbhbdev alpha merge-queue status N` prints a reason code and the next event the entry waits for | 3 |
| Dry runs | A repository in shadow mode builds candidates and logs what it would merge without merging. `status N --dry-run` says whether a pull request would be admitted | 2 |
| Approval at the exact head | Admission checks that an approval's `commit_id` equals the head, and the merge call passes that head as its guard. A push between the two makes GitHub refuse the merge | 2 |
| Message rules | Every merge commit gets the message `tbhbdev alpha merge` builds, with Conventional Commit title, plain-text body, and trailer paragraph checks, plus `Merged-by` | 2 |
| Pause and stop | Pause per repository or for all, recorded with who set it. The operator also has a stop that needs nothing from the queue ([appendix F](F-architecture.md#the-operators-stop)) | 2 |

## Reason codes

`merge-queue status N` and the check run use one set of codes. Each code names the next event that clears it.

| Code | Meaning | Clears on |
| --- | --- | --- |
| `not_labeled` | Not in the queue | the `merge-queue` label |
| `held` | The `hold` label is set | the label's removal |
| `paused` | A pause covers this repository, with who set it | resume |
| `approval_not_at_head` | No approval covers the current head | an approval at the head |
| `checks_not_green` | A check at the head failed or is still running, named in the output | a green check suite |
| `threads_open` | Unresolved review threads | resolution |
| `conflict` | The head doesn't merge cleanly onto the projected main | a new head |
| `message_invalid` | The title or trailer paragraph fails `alpha merge`'s rules | an edit to the pull request |
| `waiting_parent` | A stacked parent hasn't merged | the parent's merge |
| `waiting_dependency` | A `Depends-on` pull request hasn't merged | its merge |
| `queued` | Admitted, with position and the group ahead | a build slot |
| `building` | In a group, with the job, step, and elapsed time | CI completion |
| `removed` | The last group failed because of this entry, with the failed check | a new head and a new label |
