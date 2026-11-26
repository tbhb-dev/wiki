---
type: Design Draft
title: One public monorepo for the factory
description: Moves tbhb-dev's factory repositories into one public Bazel monorepo with a thin private companion, and says how agents, review, and CI work on it.
tags: [monorepo, bazel, git, ci, agents, design]
status: draft
owner: Design author
updated: "2026-11-26"
decisions_open: 0
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo bzlmod cutover tbhbdev -->

# One public monorepo for the factory

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Design author | 2026-11-26 | Nothing. See [decisions.md](decisions.md#decided) |

## Summary

This design covers 24 repositories. In the 28 days to 2026-11-25, 1,908 changes landed across the 22 measured repositories, including 492 dependency updates. Up to 268 of those updated an internal tbhb-dev pin without other changes.

One public Bazel repository houses the factory, with a thin private companion for private values. Agents work in sparse Git worktrees cut from one mirror per host. One workflow tests the affected targets. GitHub queues changes before merging them.

No design questions remain open. Before publishing each private-origin import, two independent agent runs must clear it. The operator has no sign-off duty.

## Proposal

When this is built, these parts exist:

- **The monorepo.** One new public repository, working name `tbhb-dev/tbhbdev`, holds the factory under `factory/`, operations under `ops/`, shared code under `lib/`, prose under `docs/`, and room for products under `products/` (new). The command-line tool is a project at `factory/cli`. Each factory project is one folder directly under `factory/`, with an `OWNERS` file that names its owners and describes it, and its own `docs/` folder for its needs, requirements, and knowledge articles. [Appendix C](C-layout-and-ownership.md) has the tree.
- **The private companion.** One new private repository mirrors the public paths. It holds the ops inventory and host variables in `ops/`, plus the operator-account folder, private memories, and private prose (new). It depends on the monorepo, and the monorepo's files never name it. [Appendix D](D-public-private.md) has the split.
- **One Bazel workspace, with one dependency root per language.** One root module, one manifest and one lock file for each language, and build files that gazelle writes for Go and Python (new). One Renovate run keeps the roots current. [Appendix H](H-dependencies-and-renovate.md) has both.
- **Layering that the build enforces.** Bazel visibility and one query test keep products off factory internals, the factory off products, and `lib/` off both (new).
- **Separation of duties.** Build, review, approve, and merge stay apart, and the design names the identity or the App that carries out each: `tbhb`, `tbhb-agent`, the reviews App, and the tier 0 workflow. The author never approves their own change, an independent reviewer judges the exact head, and the changed paths set the review tier (existing tiers, new path map).
- **Agent workspaces.** `tbhbdev alpha agent exec` keeps one partial bare mirror per host and gives each run a sparse worktree limited to its target paths and their Bazel dependencies (new). Today it copies a full clone per run (existing). [Appendix E](E-agent-runs.md) answers the operator's direct question about large-repository runs.
- **One CI workflow.** It computes affected targets with `bazel-diff`, tests them against the remote cache on GitHub's standard hosted runners, and reports one `conclusion` check (new). A pull request that changes only a project's records runs only that project's record check. GitHub's merge queue runs the same workflow before each merge (existing on three repositories today).
- **An import tool.** `tbhbdev alpha monorepo import` brings each repository in as one copy commit that puts its files where the layout wants them and names the source commit (new). No history comes along, and the archived original keeps it.
- **Issues in two repositories.** The main private Project holds issues from both repositories, and the public roadmap holds the monorepo's public subset. Every issue carries one area label from the same monorepo-derived set. [Appendix I](I-issues-and-projects.md) has the labels and the pass over existing issues.

Work stopped on the [design for an in-house merge queue](../merge-queue/README.md). Mark it superseded once the private-origin imports merge. Factory changes then land through GitHub's queue or the companion's up-to-date check (D7).

## How it works

### The tree and its rules

Folders group files by project. Each factory project has its own folder directly under `factory/`. The design reserves `products/<name>/` for future products. Operations use `ops/`, with runner-smoke at `ops/runner-smoke` and the Ansible collections at `ops/ansible/collections`.

Bazel visibility and layering queries enforce these rules.

1. A product’s dependencies come from `lib/`. Its build files load the build rules in `lib/rules` as `@tbhb_rules` labels, and it reaches any other published module through a `lib/` project.
2. Nothing under `factory/` or `lib/` depends on a product.
3. `lib/` depends only on `lib/` and third-party code.
4. The factory publishes nothing for use outside `factory/`. Anything products need goes in `lib/`, including the shared Renovate presets at `lib/renovate-config`. Code used by standalone repositories also goes in `lib/`.
5. No Bazel target depends on `ops/`. The recommendation is for `ops/` to use its own deployment tools outside Bazel.

Each project's targets default to visibility inside that project. A gazelle directive at the top of each tree sets the default, and a query test fails when any path crosses a layer.

### Approval and review

tbhb-dev keeps four duties apart in every change, and an identity or an App carries out each one. There are two identities: `tbhb`, the operator's account, and `tbhb-agent`, the account agent runs work under.

| Duty | Who carries it out |
| --- | --- |
| Build | `tbhb-agent` in an agent run, `tbhb`, or the Renovate App |
| Review | `tbhb-agent`, in a fresh run that isn't the build run |
| Approve | The reviews App at tiers 1 and 2, and the tier 0 workflow at tier 0 |
| Merge | `tbhb-agent` or `tbhb`, through GitHub's merge queue |

The author never approves their own change. The reviewer is a fresh run, independent of the author, and uses another provider when an agent built the change. A verdict covers only the exact head it judged. Each project's `OWNERS` file names its owners and describes the project, and it grants no approval.

The review tiers are the three that the wiki's first decision record sets, unchanged. The diff sets the tier, a reviewer may raise it, and the heaviest tier among the changed paths applies. The root `AGENTS.md` carries a tier 2 path map in the form the rules repository uses today, and the map is separate from area labels.

GitHub Apps can't hold different permissions per folder, so each agent App reaches every path of the monorepo (per docs). Path limits move from repository boundaries to the main ruleset: the `conclusion` check and the independent approval. D2 records the decision in [decisions.md](decisions.md#decided).

### Public and private

The public build passes with no companion present. A stub module in `third_party/` stands in for the companion and holds fake values that match the real schema. Bazel consumers of companion values pass `--override_module` to swap in the real checkout. Under the ops recommendation, its deploys read the companion's `ops/` directly.

The public repository builds and tests. Any deploy that needs inventory or host values runs from the companion's workflows on private runners, at a pinned monorepo commit. Secrets stay in 1Password, as today.

Pushing a branch to a public repository publishes it. Host hooks scan every commit for secrets and private names before the push, and CI repeats both scans. The protected environment restricts access to the name scan secret to the base branch workflow. The workflow scans fork changes at the queue candidate.

### An agent run

1. The launcher fetches `main` into the host's bare mirror under a lock.
2. It asks Bazel, in a full checkout that only the launcher uses, which folders the run's targets and their dependencies need.
3. It adds a worktree with no checkout, sets a cone-mode sparse checkout to those folders, and checks out.
4. The agent widens the cone with `tbhbdev alpha agent widen` when it needs more, and the run record keeps each widening.
5. A reviewer's run computes its cone from the pull request's changed paths with the same function, then adds direct dependents.

Worktrees share one object store, one Bazel disk cache, and one repository cache per host. A fixed pool of worktree paths keeps the count of Bazel output bases bounded. One agent host holds 598 output bases and 297 leftover run clones for one repository today (proven, [appendix A](A-evidence.md)).

### CI and the merge queue

A pull request runs one workflow. A plan job hashes the build graph at the base and the head and lists the affected targets. Test jobs run those targets, a coverage job checks 100% branch coverage across each affected project's test targets combined, and repository checks run beside them.

A path check comes before the plan. When a diff touches only needs, requirements, or knowledge articles in projects covered by the record lane, the workflow runs their record checks and skips the build and test. Any other file sends the change through the full lane, and `conclusion` stays the one required check.

The merge queue runs the same workflow on each candidate. A push to `main` runs every target, which also fills the remote cache and saves the graph hashes for later plans. Standard hosted runners cost nothing on a public repository (per docs), so the design uses nothing else.

A change to the build rules affects every target. Sharding across runners handles that first. Remote execution comes later, only on owned hosts or with the operator's approval of a paid service ([appendix F](F-ci.md)).

### Import and cutover

The scaffolding of both repositories comes first, then `lib/renovate-config`, which brings Renovate, then rules, then `factory/cli`, prose, and services. The four private-origin repositories come last. Neither new repository ever calls or pins an old one, and a repository that hasn't moved keeps the pins it holds.

Each import sorts the old repository's issues, freezes it, and lands one pull request with one copy commit. The old repository is archived once it has no open issue and no open pull request. [Appendix G](G-import-and-cutover.md) has the order, the per-repository steps, the before and after numbers, and the organization rename that ends the migration after a passing rehearsal.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| Private content reaches the public repository | A host name, private name, or secret is public for good | Copy imports with no history, a scan on a host that holds the private-name list, two independent agent reviews of each private-origin import before push, and host hooks before every push | new |
| One queue needs to handle about 68 landings a day | At today’s 1,400 seconds per pass, a serial queue can finish 61 passes a day | Affected-target CI shortens passes, the queue tests five candidates at once, and plan step 11 measures before the next import | proven count, unverified fix |
| The rules change often, and each change affects every target | 306 rules changes in 28 days each trigger a full run | Remote cache, sharding by project, and a nested workspace so rules-only tests stay small | proven count, unverified cost |
| Nested rules modules don't test cleanly from the root | The rules' own tests need a second Bazel invocation | Plan step 2 spikes it, and the fallback runs Bazel inside the nested workspace | unverified |
| Sparse worktrees confuse gazelle or Bazel | A run writes wrong build files or can't build | Plan step 2 spikes it, and a `--full` flag checks out everything | unverified |
| A shared object store lets one run damage another's refs | A run loses work it has not pushed or reads a forged ref | Reviews check out by commit hash from GitHub, the launcher alone prunes, and copy clones remain as a launcher mode | new |
| Agent traffic passes GitHub's recommended push, merge, and read rates | GitHub throttles pushes, merges, or fetches | Grouped merges, one push for each review round, one launcher fetch at a time for each host, and a retry with backoff ([appendix A](A-evidence.md#githubs-repository-limits)) | per docs, unverified rate |
| The private-name scan can't read its secret on a queue candidate | A name that the pull request head lacked reaches `main` through the merge with `main`, and the host hooks alone catch it. This is security debt until the spike passes | The secret stays in a `main`-only environment, the secret-backed scan covers the pull request head, and plan step 2 tests scanning a candidate from the base branch workflow (D6) | unverified |
| An import carries a reference to an old repository | The new repository depends on an archive | The `import` job fails on any executable reference | new |
| The organization rename breaks a reference that doesn't redirect | Standalone CI, API calls, or cloud tokens fail | A rehearsal on a throwaway organization, a placeholder on the old login, and the checks in appendix G | per docs, unverified |
| A red `main` blocks every change | All landings stop until the fix lands | The merge queue tests each candidate on the projected `main`, and the full run on `main` opens an incident when it fails | existing mechanism |
| Fork pull requests get no cache token | Outside contributions run slowly | They run uncached on hosted runners and never reach self-hosted ones | per docs |
| CI minutes before the move are unmeasured | The design can't prove a saving | Plan step 1 records 28 days of runs before anything moves | unverified |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | The baseline: workflow runs, billed minutes, wall time, and pin-bump counts per repository for 28 days | The numbers sit in this design's appendix A with the commands that produced them | `tbhb-agent` |
| 2 | A spike in the spikes repository: copy rules and tools into a scratch repository and test the six unverified items in appendix A. The spike also tests whether the base branch workflow can scan a queue candidate with the private-name secret (D6) | The spike's README states pass or fail for each of the six and for the candidate scan | `tbhb-agent` |
| 3 | Prerequisites in the old repositories: gazelle and per-package targets in tools, and each test target inside Bazel's `medium` timeout. Go port pull requests for the front, dispatch, and plugin plumbing go to the operator before merge | `bazelisk run //:gazelle -- -mode=diff` prints nothing in each repository, and no test target sets `timeout = "long"` | `tbhb-agent` |
| 4 | The scaffolding of the monorepo and the companion: dependency roots, the inline CI workflow, a folder, a `README.md`, and an `OWNERS` file for every planned project, the tier 2 path map, the area labels, both Projects, the layering test, and the rulesets without the queue rule | `bazelisk test //...` passes on `main`, the ruleset read matches the checked-in file, both repositories' `area/` labels match the full set from the monorepo tree, and the `OWNERS` test passes | `tbhb-agent`, under the organization admin grant for repository creation |
| 5 | Tracker issues sorted using appendix I, with each retained issue imported and added to its Projects | A fresh run confirms each close list, and the sync check passes for area labels and Project membership | `tbhb-agent`, in separate sorting and confirmation runs on different providers |
| 6 | The import tool, its `import` check, the private-origin pre-push gate, and the archive gate | A scratch import passes the check, a changed file or an old-repository pin makes it fail, and a missing or stale private review verdict blocks a scratch push | `tbhb-agent` |
| 7 | renovate-config imported into `lib/renovate-config`, with the root Renovate workflow running | Renovate opens a grouped update in the monorepo, and a search for old-repository callers and pins returns nothing | `tbhb-agent` |
| 8 | Rules imported, with the lint gates | `bazelisk test //...` passes on `main` with the rules loaded from the tree, and the root `AGENTS.md` holds the rewritten tier 2 rows | `tbhb-agent` |
| 9 | Tools imported into `factory/cli`, then the launcher's mirror, worktree, cone, and cleanup changes, with a tier 2 pull request that amends [decision record 0025](../../adrs/0025-clone-main-worktrees-stay-clean-and-on-main.md) for the launcher's worktrees (D8) | Two runs on one host share one mirror, a finished run leaves no worktree after the reaper runs, and the amendment has merged | `tbhb-agent` |
| 10 | The merge queue rule on `main`, and affected-target CI as the required path | A queued pull request merges through a `merge_group` run that tests fewer targets than `//...` | `tbhb-agent` |
| 11 | Public repositories imported in the order in appendix G, including github-metrics, bazel-r2-cache, go.tbhb.dev, and the Ansible collections, with issues sorted and moved and queue wait and CI time measured after each | Each old repository has no open issue or pull request and is archived, and appendix A holds the measurements | `tbhb-agent`, with separate sorting and confirmation runs (D34) |
| 12 | Private-origin repositories split, scanned, reviewed before push, and imported in one copy commit each with no history. After these imports merge, mark the design for that queue superseded (D7) | The companion stores each complete report, passing scans and comparison, and two clear verdicts for the exact import commit and report before the push | `tbhb-agent`, with two independent tier 2 review runs |
| 13 | The after numbers, 28 days after the last import | Appendix A holds both columns of the before and after table | `tbhb-agent` |
| 14 | The organization rename as the last migration step, after a passing rehearsal | Every check in appendix G's rename section passes, and a search of both repositories for the old login returns nothing | `tbhb-agent`, under the organization admin grant |

Step 12 requires two independent agent reviews to clear each private-origin copy commit before publication. Scaffolding and imports proceed without operator sign-off. Steps 1 to 3 change nothing that's hard to undo, and they pay off even if the move stops there.

The owner column names the identity that carries a step out, as the operator ruled for this design. `tbhb-agent` commissions each step's build in an agent run, and the step's state is its pull requests and the tracker row made from this plan.

## Appendices

- [Decisions](decisions.md) records the answers and rules, including the decisions of 2026-11-26.
- [A. Evidence](A-evidence.md): what the repository reads, counts, and docs show, labeled proven, existing, new, or unverified.
- [B. Alternatives considered](B-alternatives.md): Scalar, virtual filesystems, copy clones, a single root module, and the other options not taken.
- [C. Layout and ownership](C-layout-and-ownership.md): the tree, where each repository goes, the `OWNERS` files, the dependency rules, who carries out each duty, the review tiers, and how Apps and rulesets map.
- [D. Public and private](D-public-private.md): what the companion holds, who carries out each duty in each repository, how public code reads private values, and the scan rules.
- [E. Agent runs](E-agent-runs.md): mirrors, worktrees, cones, widening, disk use, cleanup, caches, and the reviewer's view.
- [F. CI](F-ci.md): the workflow's jobs, the record lane, affected targets, coverage, runners, remote execution, and the merge queue.
- [G. Import and cutover](G-import-and-cutover.md): the prerequisites, the order, the per-repository steps, what the move removes, and the organization rename.
- [H. Dependencies and Renovate](H-dependencies-and-renovate.md): one root per language, the one-version rule, extracting a product, and the single Renovate run.
- [I. Issues and Projects](I-issues-and-projects.md): the two Projects, area labels, the issue rules, the relevance pass, and the archive gate.
