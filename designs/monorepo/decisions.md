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

<!-- cspell:words monorepo bzlmod gomodscan determinator evals avirt afenc tbhbdev cutover -->

# One public monorepo for the factory decisions

Back to [the design](README.md).

## Open

### D1. Do github-metrics, bazel-r2-cache, and go.tbhb.dev join, while awrel, gomodscan, and the Ansible collections stay standalone?

- **Recommendation:** yes.
- **Why:** the first three serve only the factory, so they gain from one build and lose nothing. awrel, gomodscan, and the Ansible collections are tools that other people install by a public path and a version, like the four standalone open source tools.
- **If yes:** the three join in plan step 11 at the paths in [appendix C](C-layout-and-ownership.md#where-each-repository-goes). awrel keeps its own repository, releases, and pins.
- **If no:** name the repositories to flip. Bringing awrel in puts its packages under the monorepo's one Go module, so its install path and its version tags change for every outside user.
- **Reversible:** yes. A repository can join later by the same import, and a folder can leave by the extraction in [appendix H](H-dependencies-and-renovate.md#extracting-a-product).

### D2. May each agent App reach every path of the monorepo?

- **Recommendation:** yes.
- **Why:** a GitHub App's permissions apply to a whole repository (per docs). Today an App installed on only some repositories can't touch the others, and one repository removes that limit. Push rules that restrict file paths don't exist for public repositories (per docs).
- **If yes:** path limits rest on the main ruleset: the `conclusion` check, and an independent approval at the tier the changed paths set ([appendix C](C-layout-and-ownership.md#separation-of-duties)). An agent can still push a branch that touches any path, and it can't merge one without both.
- **If no:** sensitive folders, such as agent configuration and ops, stay in their own repositories, and the pins between them stay.
- **Reversible:** yes, by extracting a folder to its own repository.

### D3. May agent runs on one host share one writable Git object store?

- **Recommendation:** yes.
- **Why:** the launcher copies a full clone per run today, on purpose, so runs share no objects. Worktrees from one mirror share objects and refs, which makes a run cost its sparse files instead of the whole history. The monorepo's objects are all public, so the exposure is to a run that changes another run's refs, not to a run that reads them.
- **If yes:** reviews check out a commit hash fetched from GitHub and never a local branch name, the launcher alone prunes and collects garbage, and an extended reference-transaction hook refuses updates to another run's branch. A hook can't stop a process that bypasses Git, so a run's commits that aren't pushed yet have no second copy. The companion keeps copy clones, because its objects are private.
- **If no:** each run gets a partial, sparse copy clone from a full host mirror. That keeps today's isolation and the sparse view, and each run copies every commit and tree, which grows with history ([appendix B](B-alternatives.md#partial-copy-clones)).
- **Reversible:** yes. The launcher keeps both modes behind one setting.

### D4. Does the operator sign off each private-origin import before it's pushed?

- **Recommendation:** yes, once each for ops, agent-config, agent-memories, and sm.
- **Why:** publication is a one-way door. Each import is one copy commit with no history, so the operator reads a file list and a scan report instead of a history.
- **If yes:** the import tool writes the report on a host that holds the private-name list, and the push waits for the sign-off in the companion. The report lists every public file, every finding, and every file held back.
- **If no:** the two tier 2 review runs that `tbhb-agent` starts on the import pull request read the report in the operator's place, and the operator learns of a leak after the fact.
- **Reversible:** no. A pushed commit counts as published even when it's deleted later.

### D6. May the public repository hold the private-name list as an Actions secret?

- **Recommendation:** yes.
- **Why:** host hooks are the first barrier, and a host with a broken hook has none. A CI job that scans each pull request and each queue candidate catches that case before `main`. The list itself is private, so the job needs it as a secret.
- **If yes:** the list is stored once as a repository secret, which takes the organization admin grant. The scan job prints only a file and a line, never the match. Fork pull requests get no secret, so the queue candidate is their scan.
- **If no:** scanning stays on hosts only. A leak pushed from a host with a broken hook is found by a person or not at all.
- **Reversible:** yes. Delete the secret and the job.

### D7. Does work on the in-house merge queue stop now?

- **Recommendation:** yes, and its design is marked superseded once plan step 12 lands.
- **Why:** that queue exists for private repositories that GitHub's queue refuses. This move puts their code in a public repository and gives the companion an up-to-date check instead of a queue. The operator's decisions D1, D2, D3, and D17 in that design then cover nothing.
- **If yes:** ops and agent-config merge without a queue until their imports, as they do today. Nobody creates the queue's App or its Temporal workflows.
- **If no:** the queue ships for ops and agent-config and serves them for the weeks until plan step 12.
- **Reversible:** yes. The design stays in the repository and can resume.

### D8. May the worktree decision record change to fit a bare mirror?

- **Recommendation:** yes.
- **Why:** the worktree record says a clone's main worktree stays clean and runs get worktrees under it, and a bare mirror has no main worktree. The record is the operator's standing rule. The review tier record needs no change, because D31 applies it as written.
- **If yes:** one tier 2 pull request amends the record, and worktrees live under the launcher's state directory.
- **If no:** the launcher keeps a full clone beside the mirror only to satisfy the record's wording.
- **Reversible:** yes, by a later amendment.

### D9. May an import pull request skip the size limit, with line-by-line review only of the files the import wrote or changed?

- **Recommendation:** yes, when the import check passes.
- **Why:** an import adds a whole repository, far past 1,500 lines and past the 300 files that GitHub's diff view shows, and the size check has no override. The copied files passed review where they were written. A required check proves each one is byte-identical to its source at the named commit, so nothing rides along.
- **If yes:** the size check exempts a pull request that holds one import commit with a green import check. The reviewer reads the path map, the dropped list, and every rewritten or generated file, from a local checkout.
- **If no:** no import splits, because D10 makes each one a single copy commit. An import over the limit waits until its source is cut down in the old repository to fit one commit, or until the operator names another path. Tools alone has about 105,000 lines of Python, so it can't import under this answer.
- **Reversible:** yes. Remove the exemption after the last import.

### D24. Is the organization renamed from `tbhb-dev` to `tbhbdev` as the last migration step?

- **Recommendation:** yes, as the last step, and only after the rehearsal in [appendix G](G-import-and-cutover.md#the-organization-rename) passes.
- **Why:** the repository then reads `tbhbdev/tbhbdev`, which matches the tool. After the imports, few places still spell the old login, so the last step is the cheapest moment the rename will ever have. The new login read as unclaimed on 2026-11-25 (the brief says).
- **If yes:** `tbhb-agent` carries out the step under the organization admin grant, and `tbhb` answers for it through this decision. The step is the rehearsal, the rename, a free placeholder organization on the old login at once, and then the reference classes in appendix G.
- **If no:** the organization stays `tbhb-dev`, the repository stays `tbhb-dev/tbhbdev`, and plan step 14 is dropped.
- **Reversible:** partly. The login can go back while the placeholder holds the old one, and every reference then changes twice. The old login is open to anyone between the rename and the placeholder's creation.

### D25. Does the relevance pass sort both issue sources by the criteria in appendix I?

- **Recommendation:** yes.
- **Why:** the criteria decide which work is dropped, and a wrongly closed issue is lost once its repository is archived. The two sources are the operator's tracker files and the open GitHub issues of each moved-in repository, and [appendix I](I-issues-and-projects.md#the-relevance-pass) proposes one set of criteria for both.
- **If yes:** each issue from either source is sorted into close, public, or private by that table. Each close carries a one-line reason, and each kept issue carries one area label.
- **If no:** name the criteria, for one source or for both.
- **Reversible:** yes for a wrongly kept issue, which closes later. A wrongly closed issue reopens only until its repository is archived.

### D26. Does the private Project hold issues from both repositories?

- **Recommendation:** yes.
- **Why:** the public roadmap Project holds only the monorepo's issues, by the operator's decision. One private Project that holds both gives one board for all work, and the roadmap stays the public subset of it.
- **If yes:** the issue commands add every new issue to the private Project, and every monorepo issue to the roadmap as well.
- **If no:** the private Project holds only the companion's issues, and reading all work takes two boards.
- **Reversible:** yes. Add or remove the items with one command.

### D34. Does `tbhb-agent` sort the issues from both sources, with a fresh `tbhb-agent` run confirming each close list?

- **Recommendation:** yes.
- **Why:** triage decides what work survives, so the operator names who does it. A close needs the same separation as a merge: whoever proposes a close list doesn't confirm it. This design assigns the duty to no one until the operator answers.
- **If yes:** `tbhb-agent` sorts the tracker files and each moved-in repository's open issues, in an agent run. A fresh `tbhb-agent` run that shares no context with the first, on another provider, confirms each close list before anything closes.
- **If no:** name the identity that sorts and the identity that confirms, for each source. If `tbhb` confirms each close list, each archive waits on the operator's time.
- **Reversible:** yes. A later list can go to another identity. A close already made follows D25.

### D35. May D14's product rule change to `lib/` only?

- **Recommendation:** yes.
- **Why:** the third review found that appendix C lets a product name a published package directly, against the `lib/`-only boundary the tree states. D14 as decided says "`lib/` and published modules only", and only the operator changes it.
- **If yes:** D14's sentence reads "Products depend on `lib/` only and are built with the rules." A product reaches any other published module through a `lib/` project, and the fifth layering query enforces it.
- **If no:** D14 stands. A product may name a published module, and the fifth query checks `third_party/` only.
- **Reversible:** yes, by a build file and test change.

### D36. May D28's label rule change to one identical set in both repositories?

- **Recommendation:** yes.
- **Why:** the third review found that labels taken from each repository's direct children can differ between the two. D28 as decided has each repository compute labels from its own tree, and never from an `OWNERS` file.
- **If yes:** D28's sentences read "A project is a folder under `factory/`, `products/`, `lib/`, or `docs/` that holds an `OWNERS` file. The monorepo computes the set from its tree, and the companion creates the same set from the commit it pins." Nothing inside an `OWNERS` file becomes a label.
- **If no:** D28 stands. Each repository lists its own direct children, and the companion adds a label for each folder only it has.
- **Reversible:** yes. Change the generator and relabel.

## Decided

### D33. Project records live in each project's own docs folder, and a record pull request takes a lighter CI lane

- **Decided:** 2026-11-25, by the operator.
- **Rule:** needs, requirements, and knowledge articles live in each project's own `docs/` folder, such as `factory/tbhbdev/docs/needs`, `factory/tbhbdev/docs/requirements`, and `factory/tbhbdev/docs/kb`, and not in the top-level `docs/`. A pull request whose diff touches only those paths is a record pull request, and a path check decides that. It runs only the Bazel record-check targets of the projects it touches, not the full build and test. The scans that guard what a pull request publishes still run, and the private-name scan among them runs only if D6 is yes, which is this design's reading and not the operator's wording. Any other file in the diff, or a change to a check's definition, sends it through the full lane. `conclusion` stays the one required check, and the rulesets don't change ([appendix F](F-ci.md#the-record-lane)).
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

### D23. The command-line tool lives at `factory/tbhbdev`

- **Decided:** 2026-11-25, by the operator.
- **Rule:** tools imports to `factory/tbhbdev`, and its area label is `area/factory/tbhbdev`. No folder sits between `factory/` and the tool.
- **Undo:** one folder move, which renames the area label and every Bazel label below it. It costs nothing before the tools import.
- **Record:** the second fix brief for this design's pull request, 2026-11-25.

### D29. Go port pull requests for the front, dispatch, and plugin plumbing go to the operator before merge

- **Decided:** 2026-11-25, by the operator.
- **Rule:** a Go port pull request that touches the tool's front, its dispatch, or its plugin plumbing merges only after the operator has read it. Scaffolding pull requests and import pull requests carry no such gate, and D4 still covers private-origin imports.
- **Undo:** the operator lifts the gate.
- **Record:** the fix brief for this design's pull request, 2026-11-25.

### D28. Issues live in two repositories, with two Projects and one area label each

- **Decided:** 2026-11-25, by the operator. The label rule is the operator's later ruling of the same day, which replaced labels generated from owner files. Labels still come from the tree and never from an `OWNERS` file.
- **Rule:** issues live in the monorepo and in the companion, and both have Issues and Projects on. A public roadmap Project holds only the monorepo's issues, and a private Project exists beside it. Each issue carries exactly one area label: `area/<top>/<name>` for a direct child of `factory/`, `products/`, `lib/`, or `docs/`, or one of `area/repo`, `area/third_party`, and `area/none`. Each repository computes its labels from its own tree on every push to `main`, and the companion adds the public set, so no private project's name becomes a public label. The issue commands and a sync check enforce the one-label rule. Existing issues get a relevance pass before they move ([appendix I](I-issues-and-projects.md)).
- **Undo:** change the label generator or the Projects. Imported issues stay where they landed.
- **Record:** the operator's brief of 2026-11-25, addendum 5, and the second fix brief for this design's pull request.

### D27. A clean break, starting with scaffolding, then renovate-config, then rules

- **Decided:** 2026-11-25, by the operator.
- **Rule:** neither new repository calls, pins, or extends an old repository, and their CI is written inline. The import order is the scaffolding, then renovate-config, which brings Renovate, then rules, then the rest. A repository that hasn't moved yet keeps the pins it holds until its own import.
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
- **Rule:** the 13 rules modules keep their own `MODULE.bazel` files under `factory/rules`, and the root loads them through `local_path_override`. Labels such as `@tbhb_rules//lint:defs.bzl` don't change. A standalone repository can load them through `git_override` with a `strip_prefix` at its own cutover. A test fails when a nested module declares a version that differs from the root's.
- **Undo:** fold them into the root module, which rewrites every load statement and cuts off standalone consumers.
- **Record:** this design's pull request.

### D14. Five top-level folders, a flat `factory/`, and three dependency rules

- **Decided:** 2026-11-25, by the operator. The flat `factory/` is the operator's later ruling of the same day, which replaced the first sketch's extra level of folders.
- **Rule:** top-level folders are `factory/`, `products/`, `lib/`, `docs/`, and `third_party/`. `factory/` is one boundary with no grouping folders: each project is one folder directly under it, and one source repository becomes one folder. Each project's `README.md` and `OWNERS` file describe the project, and neither sorts it into a category. The top-level `docs/` holds the wiki, research, and spikes, and each project's own records live in the project (D33). Products depend on `lib/` and published modules only, the factory never depends on a product, and Bazel visibility enforces both. The companion mirrors the public paths.
- **Undo:** move folders by pull request. The rules live in build files and one test.
- **Record:** the operator's brief of 2026-11-25, and the second and third fix briefs for this design's pull request.

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

- **Decided:** 2026-11-25, by the operator. That the graph repository stays out is the operator's later ruling of the same day, which took it out of D1.
- **Rule:** tools, agent-config, agent-memories, ops, wiki, research, spikes, event-gateway, rules, dotfiles, evals, dashboard, design, agent-capacity, runner-smoke, pm, sm, renovate-config, telemetry, and work move. afakt, avirt, afenc, and ascm stay standalone and are consumed, if at all, as Bazel modules at pinned versions. The graph repository is out of scope and doesn't move. The layout leaves room for products and names none.
- **Undo:** import or extract a repository later.
- **Record:** the operator's brief of 2026-11-25, and the fifth fix brief for this design's pull request.

### D10. One fresh public monorepo, one thin private companion, and no imported history

- **Decided:** 2026-11-25, by the operator. The import rule is the operator's later ruling of the same day, which replaced the brief's first wording.
- **Rule:** the factory moves into one new public repository, and private values go to one new private companion. Each repository comes in as one reorganizing copy commit that puts its files where the layout wants them and names the source repository and commit. No history comes into the monorepo for any repository, public or private, and the archived originals keep it.
- **Undo:** one-way door for each published commit. The move itself can stop after any import, because old repositories are archived and not deleted.
- **Record:** the operator's brief of 2026-11-25, addendum 5.

## Open items at a glance

| Item | Question | Recommendation |
| --- | --- | --- |
| D1 | Three factory-only repositories join, and awrel, gomodscan, and the Ansible collections stay out | yes |
| D2 | Each agent App reaches every path | yes |
| D3 | Runs on a host share one Git object store | yes |
| D4 | The operator signs off each private-origin import | yes |
| D6 | The private-name list is an Actions secret in the public repository | yes |
| D7 | Work on the in-house merge queue stops | yes |
| D8 | The worktree record changes to fit a bare mirror | yes |
| D9 | Import pull requests skip the size limit | yes |
| D24 | The organization is renamed as the last step | yes, after the rehearsal |
| D25 | The relevance pass sorts both issue sources by appendix I's criteria | yes |
| D26 | The private Project holds issues from both repositories | yes |
| D34 | `tbhb-agent` sorts the issues, and a fresh `tbhb-agent` run confirms each close list | yes |
| D35 | D14's product rule changes to `lib/` only | yes |
| D36 | D28's label rule changes to one identical set in both repositories | yes |
