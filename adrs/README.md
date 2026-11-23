<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Decision records

This public repository holds the decision records that can be stated without anything private. An ADR goes here whenever it can be sanitized without losing its meaning: host names, the tailnet, IP addresses, internal paths, account names and credentials are removed or made generic, and the rule still reads the same. Only a record that would lose its meaning goes in the private wiki repository's `adrs/` folder.

A decision record (ADR) states one decision as a rule in the present tense, with the context that led to it and what follows from it. A record makes sense to a reader of this repository alone.

## Layout

Each record is `adrs/NNNN-<slug>.md`: the next free number, zero-padded to four digits, then a kebab-case slug of the title, such as `0007-squash-merge-every-pull-request.md`. This repository numbers its own records from `0001`. [TEMPLATE.md](TEMPLATE.md) is the house template, and the format and writing rules are in the decision-records design.

## Rules

- A record never names a host, the tailnet, a vault, an internal path, an account, a private repository or a public IP address. It uses role labels such as "the Linux build host".
- A record never cites a Decisions note or links to a private repository.
- A record never quotes anyone or says who decided. Acceptance is the pull request's approval and merge, and the pull request cites the source of the decision.
- The `decided` field is the date of the decision, not of the record.
- Nobody renames a record after it merges, with one exception: the fix for a duplicate number renames the later file.

## Records

Add each new record's entry at the end of this list, as `[title](NNNN-slug.md)` and the record's description. When two pull requests add a record at once, the later one rebases and takes the next number.

- [Agents reach GitHub through tbhbdev commands, never gh](0001-agents-reach-github-through-tbhbdev-commands-never-gh.md): Agents never run gh or call the GitHub API directly; each GitHub operation they need is a tbhbdev command, and harness guards deny the rest.
- [Agents close pull requests with a reason and keep the branch, and the close guard fails closed](0002-agents-close-pull-requests-with-a-reason-and-keep-the-branch.md): Agents close a pull request with tbhbdev pr close and a reason, the branch stays, and the guard on gh closes blocks when it can't read approvals.
- [Pull requests merge with merge commits only](0003-pull-requests-merge-with-merge-commits-only.md): Every tbhb-dev repository merges with merge commits only, stacked pull requests work natively, and authors may force-push with lease to clean up a PR branch.
- [Approved changes merge without a time-delay wait](0004-approved-changes-merge-without-a-time-delay-wait.md): No merge waits on a timer; once a change has its approval and checks it merges, and safety comes from review, classification, cheap reverts and drift reporting.
- [Pull requests merge without rebasing, and a red main is fixed first](0005-pull-requests-merge-without-rebasing-and-a-red-main-is-fixed-first.md): A pull request needn't be up to date with main to merge; each repository's main is watched, and a red main is fixed by pull request ahead of other work.
- [Renovate pull requests merge automatically once approved and green](0006-renovate-pull-requests-merge-automatically-once-approved-and-green.md): GitHub automerge merges a Renovate pull request as soon as its required approval and checks are in, with no session's go and no owner merge.
- [A clone's main worktree stays clean and on main by mechanism](0007-a-clones-main-worktree-stays-clean-and-on-main-by-mechanism.md): A clone's main worktree stays clean and on its default branch, enforced by the run launcher, a harness guard, a git hook and a scheduled clone audit.
- [Pull request cycle time runs from the earliest start to merge or close](0008-pull-request-cycle-time-runs-from-the-earliest-start-to-merge-or-close.md): A pull request's cycle time starts at the earliest started_at on its identity trailers or its earliest commit time, and ends at merge or at close without merge.
- [tbhb-dev has one definition of done, and repositories only add to it](0009-tbhb-dev-has-one-definition-of-done.md): One definition of done, kept in tbhb-dev/rules and synced to every repository, which may add items but never drop them; acceptance criteria follow schema-checked rules.
- [Pull request and issue bodies never restate CI checks](0010-pull-request-and-issue-bodies-never-restate-ci-checks.md): Pull request bodies, issue bodies and acceptance criteria carry no claims about tests, gates, lint or coverage; CI status on the head is the record.
- [Wording-only follow-ups fold into the next pull request](0011-wording-only-follow-ups-fold-into-the-next-pull-request.md): A follow-up that changes only wording rides in the next pull request the repository needs, rather than opening alone.
