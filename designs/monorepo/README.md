---
type: Design Draft
title: One public monorepo for the factory
description: Moves tbhb-dev's factory repositories into one public Bazel monorepo with a thin private companion, and says how agents, review, and CI work on it.
tags: [monorepo, bazel, git, ci, agents, design]
status: draft
owner: Monorepo maintainer
updated: "2026-11-25"
decisions_open: 9
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo rdeps buildfiles somepath bzlmod cutover -->

# One public monorepo for the factory

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Monorepo maintainer | 2026-11-25 | 9 open decisions in [decisions.md](decisions.md), about 20 minutes |

## Summary

The factory lives in about 20 repositories that pin each other. In the 28 days to 2026-11-25, 1,710 changes landed in them, and 426 were dependency updates. Up to 216 of those only moved a pin from one tbhb-dev repository to another.

This design puts the factory in one public Bazel repository, with a thin private companion for private values. Agents work in sparse Git worktrees cut from one mirror per host. One workflow tests only the targets a change affects, and GitHub's merge queue lands it.

Nothing is built until the operator reads this and says go. The operator then answers nine questions, signs off each publication of private-origin code, and reads one repository instead of 20.

## Proposal

When this is built, these parts exist:

- **The monorepo.** One new public repository, working name `tbhb-dev/monorepo`, holds the factory under `factory/`, shared code under `lib/`, prose under `docs/`, and room for products under `products/` (new). The monorepo maintainer role owns the root. [Appendix C](C-layout-and-ownership.md) has the tree.
- **The private companion.** One new private repository mirrors the public paths and holds only private values: the ops inventory and host variables, the operator-account folder, private memories, and private prose (new). It depends on the monorepo, and the monorepo never names it. [Appendix D](D-public-private.md) has the split.
- **One Bazel workspace.** One root module, one lock file per language, and build files that gazelle writes for Go and Python (new). The rules stay separate Bazel modules inside the tree, so standalone repositories keep loading them by the same labels.
- **Layering that the build enforces.** Bazel visibility and one query test keep products off factory internals, the factory off products, and `lib/` off both (new).
- **`OWNERS` files.** Each folder names its owning role and its review tier. The review tool reads them from the base branch and routes reviews by changed path (new, extends the rule that review policy comes from the base branch).
- **Agent workspaces.** `tbhbdev alpha agent exec` keeps one partial bare mirror per host and gives each run a sparse worktree limited to its target paths and their Bazel dependencies (new). Today it copies a full clone per run (existing). [Appendix E](E-agent-runs.md) answers the operator's direct question about large-repository runs.
- **One CI workflow.** It computes affected targets with `bazel-diff`, tests them against the remote cache on GitHub's standard hosted runners, and reports one `conclusion` check (new). GitHub's merge queue runs the same workflow before each merge (existing on three repositories today).
- **An import tool.** `tbhbdev alpha monorepo import` brings each repository in under its target path, with history for public repositories and one squashed commit for private ones (new). A check proves the imported tree equals the source tree.

This design overlaps the [in-house merge queue design](../merge-queue/README.md). After the move, every factory change lands through GitHub's queue or the companion's up-to-date check, so that queue has no repository left to serve (see D7).

## How it works

### The tree and its rules

Folders go by role, then by project, and never by language. `factory/` holds the command-line tool, agent configuration, build rules, services, work tracking, evaluations, and infrastructure. `products/<name>/` is empty today and names no product.

Three rules hold, and Bazel fails the build when one breaks:

1. A product depends on `lib/` and on published modules, and on nothing under `factory/`. The build rules count as published modules, because every consumer loads them as `@tbhb_rules` labels.
2. Nothing under `factory/` or `lib/` depends on a product.
3. `lib/` depends only on `lib/` and third-party code.

Each project's targets default to visibility inside that project. A gazelle directive at the top of each tree sets the default, and a query test fails when any path crosses a layer.

### Ownership and review

The nearest `OWNERS` file up the tree names the role that owns a changed file and the tier of review it needs. A root policy file lists paths that are always tier 2, such as `.github/`, rulesets, `OWNERS` files, and harness configuration, and no folder can lower them. A change takes the highest tier among its paths.

GitHub Apps can't hold different permissions per folder, so each agent App reaches every path of the monorepo (per docs). Path limits move from repository boundaries to required checks and the main ruleset. That loosens a boundary, and D2 asks the operator about it.

### Public and private

The public build passes with no companion present. A stub module in `third_party/` stands in for the companion and holds fake values that match the real schema. A deploy passes `--override_module` to swap in the real companion checkout.

The public repository builds and tests. Any deploy that needs inventory or host values runs from the companion's workflows on private runners, at a pinned monorepo commit. Secrets stay in 1Password, as today.

Pushing a branch to a public repository publishes it. Host hooks scan every commit for secrets and private names before the push, and CI repeats the scan as a second layer.

### An agent run

1. The launcher fetches `main` into the host's bare mirror under a lock.
2. It asks Bazel, in a full checkout that only the launcher uses, which folders the run's targets and their dependencies need.
3. It adds a worktree with no checkout, sets a cone-mode sparse checkout to those folders, and checks out.
4. The agent widens the cone with `tbhbdev alpha agent widen` when it needs more, and the run record keeps each widening.
5. A reviewer's run computes its cone from the pull request's changed paths with the same function, then adds direct dependents.

Worktrees share one object store, one Bazel disk cache, and one repository cache per host. A fixed pool of worktree paths keeps the count of Bazel output bases bounded. One agent host holds 598 output bases and 297 leftover run clones for one repository today (proven, [appendix A](A-evidence.md)).

The monorepo starts at about 5,100 files and 23 MB. A full checkout is cheap today, so the sparse view is for scope and for growth. The gain that matters now is cleanup.

### CI and the merge queue

A pull request runs one workflow. A plan job hashes the build graph at the base and the head and lists the affected targets. Test jobs run those targets, a coverage job checks 100% branch coverage across each affected project's test targets combined, and repository checks run beside them.

The merge queue runs the same workflow on each candidate. A push to `main` runs every target, which also fills the remote cache and saves the graph hashes for later plans. Standard hosted runners cost nothing on a public repository (per docs), so the design uses nothing else.

A change to the build rules affects every target. Sharding across runners handles that first. Remote execution comes later, only on owned hosts or with the operator's approval of a paid service ([appendix F](F-ci.md)).

### Import and cutover

Rules and the Renovate presets come first, then the command-line tool, then prose, then services. The four private-origin repositories come last. Each import freezes the old repository, lands one pull request, moves issues, and archives the old repository with a pointer. [Appendix G](G-import-and-cutover.md) has the order, the per-repository steps, and the before and after numbers.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| Private content reaches the public repository | A host name, private name, or secret is public for good | Squash imports, a scan on a host that holds the private-name list, the operator's sign-off per import (D4), and host hooks before every push | new |
| One queue carries about 61 landings a day | A serial queue at today's 1,400 seconds per pass holds under 62 passes a day, so the queue backs up | Affected-target CI shortens passes, the queue groups up to five entries, and plan step 9 measures before the next import | proven count, unverified fix |
| The rules change often, and each change affects every target | 306 rules changes in 28 days each trigger a full run | Remote cache, sharding by project, and a nested workspace so rules-only tests stay small | proven count, unverified cost |
| Nested rules modules don't test cleanly from the root | The rules' own tests need a second Bazel invocation | Plan step 2 spikes it, and the fallback runs Bazel inside the nested workspace | unverified |
| Sparse worktrees confuse gazelle or Bazel | A run writes wrong build files or can't build | Plan step 2 spikes it, and a `--full` flag checks out everything | unverified |
| A shared object store lets one run damage another's refs | A run loses work or reads a forged ref | Reviews check out by commit hash from GitHub, the launcher alone prunes, and D3 offers copy clones instead | new |
| Imported history fails the signature rule | The import pull request can't merge | All but two commits carry signatures, D5 covers the two, and the spike tests a real import | proven count, unverified on GitHub |
| A red `main` blocks every role | All landings stop until the fix lands | The merge queue tests each candidate on the projected `main`, and the full run on `main` opens an incident when it fails | existing mechanism |
| Fork pull requests get no cache token | Outside contributions run slowly | They run uncached on hosted runners and never reach self-hosted ones | per docs |
| CI minutes before the move are unmeasured | The design can't prove a saving | Plan step 1 records 28 days of runs before anything moves | unverified |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | The baseline: workflow runs, billed minutes, wall time, and pin-bump counts per repository for 28 days | The numbers sit in this design's appendix A with the commands that produced them | Monorepo maintainer |
| 2 | A spike in the spikes repository: import rules and tools into a scratch repository, test nested modules, `bazel-diff`, sparse worktrees with gazelle, and signatures on imported commits | The spike's README states pass or fail for each of the six unverified items in appendix A | Monorepo maintainer |
| 3 | Prerequisites in the old repositories: gazelle and per-package targets in tools, and each test target inside Bazel's `medium` timeout | `bazelisk run //:gazelle -- -mode=diff` prints nothing in each repository, and no test target sets `timeout = "long"` | Each repository's owning role |
| 4 | The empty monorepo and companion: root module, lint gates, `OWNERS` tooling, the layering test, the CI workflow, and the main ruleset without the queue rule | `bazelisk test //...` passes on `main`, and the ruleset read matches the checked-in file | Monorepo maintainer, with the admin role for repository creation |
| 5 | The import tool and its tree-equality check | An import of a scratch repository passes the check, and a changed file makes it fail | Tools maintainer |
| 6 | Rules and Renovate presets imported, and their reusable workflows moved to the root | A standalone repository builds with its rules pin on a monorepo commit | Rules maintainer |
| 7 | Tools imported, then the launcher's mirror, worktree, cone, and cleanup changes | Two runs on one host share one mirror, and a finished run leaves no worktree after the reaper runs | Tools maintainer |
| 8 | The merge queue rule on `main`, and affected-target CI as the required path | A queued pull request merges through a `merge_group` run that tests fewer targets than `//...` | Monorepo maintainer |
| 9 | Public repositories imported in the order in appendix G, with queue wait and CI time measured after each | Each old repository is archived with a pointer, and appendix A holds the measurements | Each repository's owning role |
| 10 | Private-origin repositories split, scanned, signed off, and imported by squash | The scan report for each import is in the companion, and the operator's sign-off is in the import pull request | Ops maintainer and agent-config maintainer |
| 11 | The after numbers, 28 days after the last import | Appendix A holds both columns of the before and after table | Monorepo maintainer |

Steps 1 to 3 change nothing that's hard to undo, and they pay off even if the move stops there.

## Appendices

- [Decisions](decisions.md): 9 open, then the decided record.
- [A. Evidence](A-evidence.md): what the repository reads, counts, and docs show, labeled proven, existing, new, or unverified.
- [B. Alternatives considered](B-alternatives.md): Scalar, virtual filesystems, copy clones, a single root module, and the other options not taken.
- [C. Layout and ownership](C-layout-and-ownership.md): the tree, where each repository goes, the dependency rules, `OWNERS`, tiers per path, and how Apps and rulesets map.
- [D. Public and private](D-public-private.md): what the companion holds, how public code reads private values, and the scan rules.
- [E. Agent runs](E-agent-runs.md): mirrors, worktrees, cones, widening, disk use, cleanup, caches, and the reviewer's view.
- [F. CI](F-ci.md): the workflow's jobs, affected targets, coverage, runners, remote execution, and the merge queue.
- [G. Import and cutover](G-import-and-cutover.md): the order, the per-repository steps, what the move removes, and the prerequisites.
