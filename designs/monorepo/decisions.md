---
type: Design Appendix
title: One public monorepo for the factory decisions
description: The open and decided questions of the monorepo design.
tags: [monorepo, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo bzlmod gomodscan determinator evals avirt afenc tbhbdev cutover msys -->

# One public monorepo for the factory decisions

Back to [the design](README.md).

## Open

None.

## Decided

### D37. Factory projects share one documents folder

- **Rule:** Store the factory's knowledge articles in `factory/docs/kb/`, needs in `factory/docs/needs/`, and requirements in `factory/docs/requirements/`. Use this shared tree for all factory records and give no factory project a documents folder of its own. This updates D33 for the factory.
- **Undo:** Move the records to project folders and update the record lane's path check.
- **Record:** Operator ruling of 2026-11-26.

### D38. Repository-wide helpers go in `scripts/`

- **Rule:** The top-level `scripts/` holds helpers for internal use across the repository. It has no project of its own and publishes nothing. The name avoids the confusion with the old tools repository that `tools/` would cause.
- **Undo:** Rename the folder.
- **Record:** Operator ruling of 2026-11-26.

### D39. Runner-smoke becomes repository infrastructure

- **Rule:** Move all four workflows to the root `.github/workflows/` and all six sample projects to `ops/runner-smoke/`. List `ops/runner-smoke/` in `.bazelignore`. Drop the source repository's boilerplate, `msys2-install`, and the Windows legs. The fixture folder belongs to `ops/` and has no project of its own.
- **Undo:** Restore the source repository from its archive and move the smoke checks back.
- **Record:** Operator ruling of 2026-11-26.

### D40. Renovate-config workflows live at the root

- **Rule:** Move all three workflows to the root `.github/workflows/`. Keep the presets, pin script, and its tests in `lib/renovate-config`.
- **Undo:** Restore the source repository from its archive and move the configuration and workflows back.
- **Record:** Operator ruling of 2026-11-26.

### D41. Import each project before porting it or splitting its build

- **Rule:** Start with scaffolding, then import renovate-config, then rules, then the command-line tool at `factory/cli` and ops in parallel, then the rest. Import each project as it is and port to Go after import. Immediately after each project import, split its build into fine-grained targets in a separate change. Use Gazelle where a Gazelle extension exists. The import commit keeps the source build granularity so `verify` can compare the import with its source. [Appendix G](G-import-and-cutover.md)
- **Undo:** Reorder pending imports or revert a build split or port independently of its import.
- **Record:** Operator ruling of 2026-11-26.

### D42. The review-policy checker test belongs with the command-line tool

- **Decided:** 2026-11-26, by the coordinator. The decision is reversible.
- **Rule:** The test exercises the command-line tool's checker, so the wave 2 rules import records it as a drop in the `lib/rules` path map and removes the old tools pin (D27). The wave 3 import adds the test under `factory/cli` and reads the policy data from `lib/rules`, which `factory/` may depend on (D35). The check is absent from monorepo CI until the CLI import merges. The rules import's pull request body names that gap.
- **Undo:** Move the test back under `lib/rules` and allow that one dependency on `factory/cli`.
- **Record:** The coordination call of 2026-11-26, recorded in [appendix C](C-layout-and-ownership.md#where-each-repository-goes) and [appendix G](G-import-and-cutover.md#import-order).

### D7. Stop work on the in-house queue

- **Decided:** 2026-11-26, by the operator on 2026-11-25.
- **Rule:** stop work on the in-house merge queue. Once the private-origin imports in plan step 12 merge, mark the [design for that queue](../merge-queue/README.md) superseded. Until then, the source repositories keep merging as they do today.
- **Undo:** reopen the design if a repository needs that queue again.
- **Record:** the operator's decision of 2026-11-25.

### D8. The launcher keeps its mirror's run worktrees under its state directory

- **Decided:** 2026-11-26, by the orchestrator.
- **Rule:** the launcher creates run worktrees from its bare mirror and keeps them in its state directory. The existing worktree rule still applies to people's clones. Amend [decision record 0025](../../adrs/0025-clone-main-worktrees-stay-clean-and-on-main.md) by a tier 2 pull request when the launcher changes.
- **Undo:** amend the record and change where the launcher creates its worktrees.
- **Record:** the decision by the orchestrator on 2026-11-26. Plan step 9 amends the record.

### D9. Imports use the existing override for pull request size

- **Decided:** 2026-11-26, by the orchestrator.
- **Rule:** send an import that exceeds either limit to a sizing review agent under the existing rule for pull request size. The agent can override either limit for the exact head it reviewed. It uses the passing byte-identical import check for that head as evidence, with the companion's byte comparison for private-origin imports. A new head needs a new verdict. The import reviewer works from a local checkout of that head and reads the path map and every dropped, rewritten, or generated file ([appendix G](G-import-and-cutover.md#importing-one-repository)).
- **Undo:** change the import review procedure while retaining the existing size rule.
- **Record:** the decision by the orchestrator on 2026-11-26.

### D1. Four factory-only imports join, and gomodscan stays standalone

- **Decided:** 2026-11-26, by the operator.
- **Rule:** github-metrics, bazel-r2-cache, go.tbhb.dev, and the Ansible collections join in plan step 11 at the paths in [appendix C](C-layout-and-ownership.md#where-each-repository-goes). gomodscan stays standalone, and awrel stays out because it's retiring.
- **Undo:** Import a standalone repository later or extract an imported project as described in [appendix H](H-dependencies-and-renovate.md#extracting-a-product).
- **Record:** the operator's answers of 2026-11-26.

### D2. Each agent App reaches every path of the monorepo

- **Decided:** 2026-11-26, by the operator.
- **Rule:** Each agent App is installed on the whole repository. An agent can push a branch that touches any path. Merging requires a green `conclusion` and independent approval at the tier the changed paths and content set ([appendix C](C-layout-and-ownership.md#separation-of-duties)).
- **Undo:** Extract a project to a separate repository and restrict which Apps are installed there.
- **Record:** the operator's answers of 2026-11-26.

### D3. Agent runs on one host share a writable Git object store

- **Decided:** 2026-11-26, by the operator.
- **Rule:** Public runs use worktrees from the host's bare mirror. Reviews check out a commit hash fetched from GitHub. The launcher alone prunes and collects garbage, and the reference-transaction hook rejects updates to another run's branch. Direct writes to the mirror can bypass hooks, and commits a run has yet to push have no second copy. The companion keeps copy clones.
- **Undo:** Switch the launcher to partial, sparse copy clones ([appendix B](B-alternatives.md#partial-copy-clones)).
- **Record:** the operator's answers of 2026-11-26.

### D4. Independent agent reviews clear each private-origin import before push

- **Decided:** 2026-11-26, by the operator.
- **Rule:** Two independent tier 2 `tbhb-agent` review runs with private access check every public file, every finding, and every held-back file before the first public push. The import tool requires both clear verdicts for the exact copy commit and report. A missing or stale verdict, failed scan or comparison, incomplete report, or unresolved finding blocks the push. The operator has no sign-off duty. The import remains one signed copy commit with no source history ([appendix D](D-public-private.md#importing-a-private-origin-repository)).
- **Undo:** Change the gate for future imports. A public push cannot be undone.
- **Record:** the operator's answers of 2026-11-26.

### D6. The base branch workflow holds access to the private-name secret

- **Decided:** 2026-11-26, by the orchestrator.
- **Rule:** store the list in a GitHub Actions environment secret under the organization administrator grant. Put the secret in an environment whose deployment rule allows the `main` branch alone, so a `pull_request` or `merge_group` run of an agent-edited `ci.yml` can't read it. The base branch workflow reads it. It scans eligible pull requests and every merge queue candidate, including candidates with fork changes. Its logs give the file and line of each finding and omit the match ([appendix D](D-public-private.md#keeping-private-content-out)). A `merge_group` run uses the queue candidate's own workflow files and its ref isn't `main`. The environment withholds the secret from it. Plan step 2 tests whether the base branch workflow can scan a candidate. Until that passes, a candidate relies on the scan of its pull request's head and the host hooks, and the [risks table](README.md#risks-and-unknowns) records the gap.
- **Undo:** remove the environment secret and the job. Host scanning still runs.
- **Record:** the decision by the orchestrator on 2026-11-26.

### D24. The organization rename is the last migration step

- **Decided:** 2026-11-26, by the operator.
- **Rule:** After the rehearsal in [appendix G](G-import-and-cutover.md#the-organization-rename) passes, `tbhb-agent` uses the organization admin grant to rename `tbhb-dev` to `tbhbdev`, immediately claims the old login with a free placeholder organization, and checks the affected references.
- **Undo:** Restore the old login while the placeholder holds it, and update every affected reference again.
- **Record:** the operator's answers of 2026-11-26.

### D25. Both issue sources use the same relevance criteria

- **Decided:** 2026-11-26, by the operator.
- **Rule:** Apply [appendix I's relevance pass](I-issues-and-projects.md#the-relevance-pass) to the operator's tracker files and every moved-in repository's open GitHub issues. Each issue is closed, kept in public, or kept in private. Each close carries a one-line reason, and each kept issue carries one area label.
- **Undo:** Close a wrongly kept issue later. Reopen a wrongly closed issue before its source repository is archived.
- **Record:** the operator's answers of 2026-11-26.

### D26. The private Project holds both repositories' issues

- **Decided:** 2026-11-26, by the operator.
- **Rule:** The main private Project holds every issue from the monorepo and the companion. The public roadmap Project holds only the monorepo's issues. The issue commands add each new issue to the private Project and add each monorepo issue to the roadmap as well.
- **Undo:** Change Project membership and update the issue commands.
- **Record:** the operator's answers of 2026-11-26.

### D34. Separate agent runs sort issues and confirm close lists

- **Decided:** 2026-11-26, by the operator.
- **Rule:** `tbhb-agent` sorts both issue sources. A fresh `tbhb-agent` run on another provider, with no context from the first run, confirms each close list before any issue on it closes.
- **Undo:** Reassign later lists. D25 governs reopening a closed issue.
- **Record:** the operator's answers of 2026-11-26.

### D35. Products depend on lib only, and factory code stays within the factory

- **Decided:** 2026-11-26, by the operator.
- **Rule:** Products depend only on their own targets and `lib/`, which provides published modules and vendored code. Product build files load the build rules from `lib/rules` (D15). Factory projects may depend on `lib/`. The factory never depends on a product and publishes nothing for use outside `factory/`. Code that products or standalone repositories use goes in `lib/`, including the shared presets in `lib/renovate-config`. Reusable workflows stay in `.github/workflows/` ([appendix C](C-layout-and-ownership.md#where-each-repository-goes)). Bazel visibility and the layering queries in [appendix C](C-layout-and-ownership.md#dependency-rules-in-bazel) enforce both boundaries, including that no `factory/` target is visible to `products/` or `lib/`.
- **Undo:** Amend D14 and change the visibility rules and layering test together.
- **Record:** the operator's answers of 2026-11-26.

### D36. Both repositories use one identical area label set

- **Decided:** 2026-11-26, by the orchestrator.
- **Rule:** An `OWNERS` file marks a project under `factory/`, `products/`, `lib/`, or `docs/`, and at `ops/` or any folder below it (D14). The monorepo computes the set from its tree, and the companion creates the same set from the commit it pins. The generator uses the project paths and ignores the contents of `OWNERS` files.
- **Undo:** Change the generator and relabel the affected issues.
- **Record:** the decision by the orchestrator on 2026-11-26.

### D33. A record pull request takes a lighter CI lane

- **Decided:** 2026-11-25, by the operator. D37 changed the factory paths on 2026-11-26.
- **Rule:** Factory records share `factory/docs/` (D37). Other projects keep needs, requirements, and knowledge articles in their own `docs/` folders. When a pull request touches only eligible Markdown records, CI runs the Bazel record checks for the changed paths. The scans that guard publication still run, and the private-name scan runs wherever D6 supplies the secret. Any other file in the diff sends it through the full lane. `conclusion` stays the one required check ([appendix F](F-ci.md#the-record-lane)).
- **Undo:** remove the path check from the workflow, and every pull request takes the full lane again.
- **Record:** the third fix brief for this design's pull request, 2026-11-25.

### D32. Issues follow five rules that the issue commands enforce

- **Decided:** 2026-11-25, by the operator.
- **Rule:** an issue's Status changes only through the issue commands, under a state machine defined in code. One agent run holds an issue at a time, the issue shows a visible mark while it's held, and parallel work comes from Task issues. The hierarchy is Initiative, Epic, Feature, then an optional Task, and Defects, Chores, Research, and Spikes may have no parent. Several pull requests may fulfil one issue, each linked by `Refs:` or `Completes:`. An issue closes only through a guarded `issue close`, and the pull request template and body check reject GitHub's closing keywords.
- **Undo:** change the state machine or the body check by pull request.
- **Record:** the operator's brief of 2026-11-25, addendum 5.

### D31. The review tiers stay as the first decision record has them, with a tier 2 path map

- **Decided:** 2026-11-25, by the operator.
- **Rule:** this design adds no tier and renames none. Tier 0, tier 1, and tier 2 mean what the wiki's first decision record says. The diff sets the tier, a reviewer may raise it, and it never falls. The monorepo's root `AGENTS.md` carries a tier 2 path map in the form of the rules repository's "Tier 2 review paths" table, and the map is separate from area labels ([appendix C](C-layout-and-ownership.md#review-tiers)).
- **Undo:** amend the decision record by a tier 2 pull request.
- **Record:** `adrs/0001-the-changed-paths-and-content-set-the-review-tier.md`, and the second fix brief for this design's pull request, 2026-11-25.

### D30. tbhb-dev runs on separation of duties, and an identity or an App carries out each duty

- **Decided:** 2026-11-25, by the operator. The identities and the `OWNERS` files are the operator's later ruling of the same day, which replaced the second fix round's wording that dropped every owner file.
- **Rule:** build, review, approve, and merge stay apart in every change. There are two identities, `tbhb` and `tbhb-agent`, plus the GitHub Apps, and the design states which of them carries out each duty ([appendix C](C-layout-and-ownership.md#separation-of-duties)). The author never approves their own change. The reviewer is independent of the author, with a fresh context and, for an agent-built change, another provider. A verdict covers only the exact head it judged, review weight follows the tier of the changed paths, and a merge needs the checks plus the required independent approval. Each project's `OWNERS` file names its owners as identities and describes the project: what it is and what it depends on. An `OWNERS` file grants no approval, any agent run may change any path, and the monorepo has no `CODEOWNERS` file and no categories of project.
- **Undo:** change the duty table or the `OWNERS` test by a tier 2 pull request.
- **Record:** the second and third fix briefs for this design's pull request, 2026-11-25.

### D23. The command-line tool lives at `factory/cli`

- **Decided:** 2026-11-26, by the operator.
- **Rule:** The tools repository imports to `factory/cli`. The `cli` folder is the project itself and its area label is `area/factory/cli`.
- **Undo:** one folder move, which renames the area label and every Bazel label below it. It costs nothing before the tools import.
- **Record:** The second fix brief set `factory/tbhbdev` on 2026-11-25. The operator's decisions log changed the path on 2026-11-26.

### D29. Go port pull requests for the front, dispatch, and plugin plumbing go to the operator before merge

- **Decided:** 2026-11-25, by the operator.
- **Rule:** a Go port pull request that touches the tool's front, its dispatch, or its plugin plumbing merges only after the operator has read it. Scaffolding pull requests and import pull requests carry no such gate, and D4 still covers private-origin imports.
- **Undo:** the operator lifts the gate.
- **Record:** the fix brief for this design's pull request, 2026-11-25.

### D28. Issues live in two repositories, with two Projects and one area label each

- **Decided:** 2026-11-25, by the operator. D26 and D36 amend the Project and label rules on 2026-11-26.
- **Rule:** issues live in the monorepo and in the companion, and both have Issues and Projects on. The main private Project holds issues from both repositories, and the public roadmap holds the monorepo's public issues. Each issue carries exactly one area label, either `area/<project-path>` or one of `area/repo`, `area/third_party`, and `area/none`. An `OWNERS` file marks a project under `factory/`, `products/`, `lib/`, or `docs/`, and at `ops/` or any folder below it (D14). The monorepo computes the set from its tree, and the companion creates the same set from the commit it pins. The issue commands and a sync check enforce the one-label rule. Existing issues get a relevance pass before they move ([appendix I](I-issues-and-projects.md)).
- **Undo:** change the label generator or the Projects. Imported issues stay where they landed.
- **Record:** the operator's brief of 2026-11-25, addendum 5, and the second fix brief for this design's pull request.

### D27. The new repositories have no dependency on the old repositories

- **Decided:** 2026-11-25, by the operator.
- **Rule:** Neither new repository calls, pins, or extends an old repository, and their CI is written inline. A repository that hasn't moved yet keeps the pins it holds until its own import. D41 sets the import order.
- **Undo:** none needed. A stop after any import leaves the old repositories working ([appendix G](G-import-and-cutover.md#stopping-partway)).
- **Record:** the operator's brief of 2026-11-25, addendum 5.

### D5. No signature exception for dotfiles and runner-smoke

- **Decided:** 2026-11-25, by the operator's rule in D10.
- **Rule:** this item asked whether two repositories with one unsigned commit each could skip their history. Every import is now one new signed commit with no history, so the question no longer arises and no ruleset bypass exists.
- **Undo:** none needed.
- **Record:** the operator's brief of 2026-11-25, addendum 5.

### D22. Old repositories are archived once they're empty of open work, and never deleted

- **Decided:** 2026-11-25, by the operator.
- **Rule:** a moved-in repository becomes a read-only archive once it has no open issue and no open pull request. Its README gets a pointer first. Its history, closed issues, pull requests, releases, and tags stay readable there.
- **Undo:** take it out of the archive in repository settings.
- **Record:** the operator's brief of 2026-11-25, addendum 5.

### D21. No Scalar command and no virtual filesystem

- **Decided:** 2026-11-25, by the design's author.
- **Rule:** the launcher uses partial clone, cone-mode sparse checkout, and `git maintenance` from core Git directly. It doesn't use the `scalar` command, VFS for Git, or a FUSE filesystem. [Appendix B](B-alternatives.md#scalar) gives the reasons and what would change them.
- **Undo:** adopt one later. Nothing in the layout depends on this.
- **Record:** this design's pull request.

### D20. The repositories are `tbhb-dev/tbhbdev` and `tbhb-dev/tbhbdev-private`

- **Decided:** 2026-11-25, by the operator.
- **Rule:** the public monorepo's working name is `tbhb-dev/tbhbdev`, and the private companion is `tbhb-dev/tbhbdev-private`. They're a single pair, and no third repository joins them. The monorepo's files never name the companion: they reach it through the `tbhb_private` module and one host setting.
- **Undo:** rename a repository. GitHub redirects the old name for Git, and D24 covers the organization's name.
- **Record:** the operator's brief of 2026-11-25, addenda 4 and 5.

### D19. The design system goes under `lib/`, not `docs/`

- **Decided:** 2026-11-25, by the design's author.
- **Rule:** the design repository is a workspace of published packages, so it lands at `lib/design`, where products may depend on it. The starting sketch put it under `docs/`.
- **Undo:** move the folder in one pull request.
- **Record:** this design's pull request.

### D18. One dependency root per language, and one version of each dependency

- **Decided:** 2026-11-25, by the design's author, on the brief's proposal.
- **Rule:** the repository has one `MODULE.bazel`, one uv workspace, one `go.mod`, one Cargo workspace, one pnpm workspace, and one root `Package.swift`, each with one lock file that Bazel reads. There's no `go.work`, because no written case needs one. A second version of a dependency needs a dated entry in the exceptions file ([appendix H](H-dependencies-and-renovate.md)).
- **Undo:** split a root by pull request. Each split brings back one more lock file to update.
- **Record:** this design's pull request.

### D17. Standard hosted runners only, and no paid build service

- **Decided:** 2026-11-25, by the design's author, inside the operator's rule that no paid service starts without approval.
- **Rule:** the public repository's CI runs on GitHub's standard hosted runners, Linux for most jobs and macOS for Swift builds. It uses no larger runner, no paid remote execution, and no self-hosted runner on `pull_request` events. The companion keeps the private runner label.
- **Undo:** change the runner labels, after the operator approves any cost.
- **Record:** this design's pull request.

### D16. `bazel-diff` computes affected targets

- **Decided:** 2026-11-25, by the design's author.
- **Rule:** CI hashes the build graph per commit with `bazel-diff` and compares the base's hashes with the head's. The hashes for a `main` commit are saved once and reused. The other candidate, `target-determinator`, configures both revisions on every run ([appendix B](B-alternatives.md#target-determinator)).
- **Undo:** swap the plan job's tool. The test jobs take a target list from either.
- **Record:** this design's pull request.

### D15. The rules stay nested Bazel modules

- **Decided:** 2026-11-25, by the design's author.
- **Rule:** D35 places the build rules in `lib/rules` because products and standalone repositories load them. The 13 rules modules keep their own `MODULE.bazel` files under `lib/rules`, and the root loads them through `local_path_override`. Labels such as `@tbhb_rules//lint:defs.bzl` don't change. A standalone repository can load them through `git_override` with a `strip_prefix` at its own cutover. A test fails when a nested module declares a version that differs from the root's.
- **Undo:** fold them into the root module, which rewrites every load statement and cuts off standalone consumers.
- **Record:** this design's pull request.

### D14. Top-level folders, a flat `factory/`, and dependency rules

- **Decided:** 2026-11-26, by the operator. The operator set the flat `factory/` layout on 2026-11-25. D35 amended the dependency rules on 2026-11-26.
- **Rule:** The layout uses `factory/`, `ops/`, `products/`, `lib/`, `docs/`, `scripts/` (D38), and `third_party/`. `factory/` stays flat, with one folder directly under it per project and a shared `factory/docs/` (D37). The public operations files go in `ops/` and private values go in the companion's `ops/`. The Ansible collections go in `ops/ansible/collections`, and the runner-smoke samples go in `ops/runner-smoke/` (D39). Each project's `README.md` and `OWNERS` file describe the project, and neither sorts it into a category. The top-level `docs/` holds the wiki, research, and spikes. D35 governs the factory and product dependency rules. The companion mirrors the public paths.
- **Undo:** move folders by pull request. The rules live in build files and one test.
- **Record:** The operator's brief of 2026-11-25, the second and third fix briefs for this design's pull request, and the operator's decisions log of 2026-11-26.

### D13. Gazelle writes build files, coverage stays at 100% branch, and CI runs affected targets

- **Decided:** 2026-11-25, by the operator.
- **Rule:** gazelle generates build files for Go and Python, and CI fails when they're stale. Coverage is 100% branch, measured across a project's test targets combined. CI on a pull request runs only the targets a change affects.
- **Undo:** each is a CI setting.
- **Record:** the operator's brief of 2026-11-25.

### D12. GitHub's queue on the public repository, and an up-to-date check on the companion

- **Decided:** 2026-11-25, by the operator.
- **Rule:** the monorepo uses GitHub's merge queue, which the Team plan offers only on public repositories. The companion requires its branch to be up to date with `main` before a merge, and it gets no in-house queue.
- **Undo:** change either ruleset.
- **Record:** the operator's brief of 2026-11-25.

### D11. The factory repositories are in scope, and the four open source tools aren't

- **Decided:** 2026-11-25, by the operator. That the graph repository stays out is the operator's later ruling of the same day, which took it out of D1. D1 adds four imports on 2026-11-26.
- **Rule:** move `tools`, `agent-config`, `agent-memories`, `ops`, `wiki`, `research`, `spikes`, `event-gateway`, `rules`, `dotfiles`, `evals`, `dashboard`, `design`, `agent-capacity`, `runner-smoke`, `pm`, `sm`, `renovate-config`, `telemetry`, `work`, `github-metrics`, `bazel-r2-cache`, `go.tbhb.dev`, and the Ansible collections. Keep `afakt`, `avirt`, `afenc`, and `ascm` standalone. Consumers pin them as Bazel modules. The graph repository is out of scope and doesn't move. The layout leaves room for products and names none.
- **Undo:** import or extract a repository later.
- **Record:** the operator's brief of 2026-11-25, and the fifth fix brief for this design's pull request.

### D10. One fresh public monorepo, one thin private companion, and no imported history

- **Decided:** 2026-11-25, by the operator. The import rule is the operator's later ruling of the same day, which replaced the brief's first wording.
- **Rule:** the factory moves into one new public repository, and private values go to one new private companion. Each repository comes in as one reorganizing copy commit that puts its files where the layout wants them and names the source repository and commit. No history comes into the monorepo for any repository, public or private, and the archived originals keep it.
- **Undo:** one-way door for each published commit. The move itself can stop after any import, because old repositories are archived and not deleted.
- **Record:** the operator's brief of 2026-11-25, addendum 5.

## Open items at a glance

| Scope | Open items |
| --- | --- |
| Monorepo design decisions | None |
