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

### D1. Do github-metrics, graph, bazel-r2-cache, and go.tbhb.dev join, while awrel, gomodscan, and the Ansible collections stay standalone?

- **Recommendation:** yes.
- **Why:** the first four serve only the factory, so they gain from one build and lose nothing. awrel, gomodscan, and the Ansible collections are tools that other people install by a public path and a version, like the four standalone open source tools. Nobody read graph in this pass, so its place rests on the coordinator's description.
- **If yes:** the four join in plan step 11 at the paths in [appendix C](C-layout-and-ownership.md#where-each-repository-goes). awrel keeps its own repository, releases, and pins.
- **If no:** name the repositories to flip. Bringing awrel in puts its packages under the monorepo's one Go module, so its install path and its version tags change for every outside user.
- **Reversible:** yes. A repository can join later by the same import, and a folder can leave by the extraction in [appendix H](H-dependencies-and-renovate.md#extracting-a-product).

### D2. May each agent App reach every path of the monorepo?

- **Recommendation:** yes.
- **Why:** a GitHub App's permissions apply to a whole repository (per docs). Today an App installed on only some repositories can't touch the others, and one repository removes that limit. Push rules that restrict file paths don't exist for public repositories (per docs).
- **If yes:** path limits rest on the main ruleset, on the required `owners` check, and on tier 2 for the paths in the root policy. An agent can still push a branch that touches any path, and it can't merge one without those.
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
- **If no:** the ops maintainer and the agent-config maintainer sign off for their own repositories, and the operator learns of a leak after the fact.
- **Reversible:** no. A pushed commit counts as published even when it's deleted later.

### D6. May the public repository hold the private-name list as an Actions secret?

- **Recommendation:** yes.
- **Why:** host hooks are the first barrier, and a host with a broken hook has none. A CI job that scans each pull request and each queue candidate catches that case before `main`. The list itself is private, so the job needs it as a secret.
- **If yes:** the admin role stores the list as a repository secret. The scan job prints only a file and a line, never the match. Fork pull requests get no secret, so the queue candidate is their scan.
- **If no:** scanning stays on hosts only. A leak pushed from a host with a broken hook is found by a person or not at all.
- **Reversible:** yes. Delete the secret and the job.

### D7. Does work on the in-house merge queue stop now?

- **Recommendation:** yes, and its design is marked superseded once plan step 12 lands.
- **Why:** that queue exists for private repositories that GitHub's queue refuses. This move puts their code in a public repository and gives the companion an up-to-date check instead of a queue. The operator's decisions D1, D2, D3, and D17 in that design then cover nothing.
- **If yes:** ops and agent-config merge without a queue until their imports, as they do today. Nobody creates the queue's App or its Temporal workflows.
- **If no:** the queue ships for ops and agent-config and serves them for the weeks until plan step 12.
- **Reversible:** yes. The design stays in the repository and can resume.

### D8. May the decision records for review tiers and for worktrees change to fit one repository?

- **Recommendation:** yes.
- **Why:** the tier record sorts changes partly by repository type, and one repository has no types. The worktree record says a clone's main worktree stays clean and runs get worktrees under it, and a bare mirror has no main worktree. Both are the operator's standing rules.
- **If yes:** two tier 2 pull requests amend them. Tiers key on paths alone, with the same three tiers and the same triggers. Worktrees live under the launcher's state directory.
- **If no:** the monorepo keeps a path-to-type table that copies today's repository types, and the launcher keeps a full clone beside the mirror only to satisfy the worktree record.
- **Reversible:** yes, by a later amendment.

### D9. May an import pull request skip the size limit, with line-by-line review only of the files the import wrote or changed?

- **Recommendation:** yes, when the import check passes.
- **Why:** an import adds a whole repository, far past 1,500 lines and past the 300 files that GitHub's diff view shows, and the size check has no override. The copied files passed review where they were written. A required check proves each one is byte-identical to its source at the named commit, so nothing rides along.
- **If yes:** the size check exempts a pull request that holds one import commit with a green import check. The reviewer reads the path map, the dropped list, and every rewritten or generated file, from a local checkout.
- **If no:** each import splits into pull requests under the limit. Tools alone has about 105,000 lines of Python, which is over 70 pull requests, and the import is then no longer one commit, against D10.
- **Reversible:** yes. Remove the exemption after the last import.

### D23. Does the command-line tool live at `factory/tbhbdev`, and not at `factory/cli/tbhbdev`?

- **Recommendation:** yes, `factory/tbhbdev`, which is also the coordinator's recommendation.
- **Why:** every other project folder is named for its project, and a `cli/` level would hold one project. The shorter path also gives shorter Bazel labels and a shorter area label.
- **If yes:** tools imports to `factory/tbhbdev`, and its area label is `area/factory/tbhbdev`.
- **If no:** tools imports to `factory/cli/tbhbdev`, which leaves a place for a second command-line tool. The appendices write the folder as `<cli>` until this is answered.
- **Reversible:** yes. It costs nothing before the tools import, and one folder move afterward, which renames the area label and every Bazel label below it.

### D24. Is the organization renamed from `tbhb-dev` to `tbhbdev` as the last migration step?

- **Recommendation:** yes, as the last step, and only after the rehearsal in [appendix G](G-import-and-cutover.md#the-organization-rename) passes.
- **Why:** the repository then reads `tbhbdev/tbhbdev`, which matches the tool. After the imports, few places still spell the old login, so the last step is the cheapest moment the rename will ever have. The new login read as unclaimed on 2026-11-25 (the brief says).
- **If yes:** the admin role runs the rehearsal, renames the organization, creates a free placeholder organization on the old login at once, and works through the reference classes in appendix G.
- **If no:** the organization stays `tbhb-dev`, the repository stays `tbhb-dev/tbhbdev`, and plan step 14 is dropped.
- **Reversible:** partly. The login can go back while the placeholder holds the old one, and every reference then changes twice. The old login is open to anyone between the rename and the placeholder's creation.

### D25. Does the imports role run the relevance pass, by the criteria in appendix I?

- **Recommendation:** yes, with each repository's owning role confirming the close list for its own issues.
- **Why:** the imports role already refines the tracker files under the [tracker import design](../pm/E-import.md), and the owning role knows which of its issues the move makes moot. The criteria are in [appendix I](I-issues-and-projects.md#the-relevance-pass).
- **If yes:** the imports role sorts each open issue of a moved-in repository into close, public, or private, and the owning role confirms the closes before anything is closed.
- **If no:** name the role and the criteria. If the operator runs the pass, each archive waits on the operator's time.
- **Reversible:** yes for a wrongly kept issue, which closes later. A wrongly closed issue reopens only until its repository is archived.

### D26. Does the private Project hold issues from both repositories?

- **Recommendation:** yes.
- **Why:** the public roadmap Project holds only the monorepo's issues, by the operator's decision. One private Project that holds both gives one board for all work, and the roadmap stays the public subset of it.
- **If yes:** the issue commands add every new issue to the private Project, and every monorepo issue to the roadmap as well.
- **If no:** the private Project holds only the companion's issues, and reading all work takes two boards.
- **Reversible:** yes. Add or remove the items with one command.

## Decided

### D29. Go port pull requests for the front, dispatch, and plugin plumbing go to the operator before merge

- **Decided:** 2026-11-25, by the operator.
- **Rule:** a Go port pull request that touches the tool's front, its dispatch, or its plugin plumbing merges only after the operator has read it. Scaffolding pull requests and import pull requests carry no such gate, and D4 still covers private-origin imports.
- **Undo:** the operator lifts the gate.
- **Record:** the fix brief for this design's pull request, 2026-11-25.

### D28. Issues live in two repositories, with two Projects and one generated area label each

- **Decided:** 2026-11-25, by the operator.
- **Rule:** issues live in the monorepo and in the companion, and both have Issues and Projects on. A public roadmap Project holds only the monorepo's issues, and a private Project exists beside it. Each issue carries exactly one `area/<path>` label, where the path is the nearest folder with an `OWNERS` file. The labels are generated from `OWNERS`, are identical in both repositories, and are enforced by the issue commands and a sync check. Existing issues get a relevance pass before they move ([appendix I](I-issues-and-projects.md)).
- **Undo:** change the label generator or the Projects. Imported issues stay where they landed.
- **Record:** the operator's brief of 2026-11-25, addendum 5.

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

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** the launcher uses partial clone, cone-mode sparse checkout, and `git maintenance` from core Git directly. It doesn't use the `scalar` command, VFS for Git, or a FUSE filesystem. [Appendix B](B-alternatives.md#scalar) gives the reasons and what would change them.
- **Undo:** adopt one later. Nothing in the layout depends on this.
- **Record:** this design's pull request.

### D20. The repositories are `tbhb-dev/tbhbdev` and `tbhb-dev/tbhbdev-private`

- **Decided:** 2026-11-25, by the operator.
- **Rule:** the public monorepo's working name is `tbhb-dev/tbhbdev`, and the private companion is `tbhb-dev/tbhbdev-private`. They're a single pair, and no third repository joins them. The monorepo's files never name the companion: they reach it through the `tbhb_private` module and one host setting.
- **Undo:** rename a repository. GitHub redirects the old name for Git, and D24 covers the organization's name.
- **Record:** the operator's brief of 2026-11-25, addenda 4 and 5.

### D19. The design system goes under `lib/`, not `docs/`

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** the design repository is a workspace of published packages, so it lands at `lib/design`, where products may depend on it. The starting sketch put it under `docs/`.
- **Undo:** move the folder in one pull request.
- **Record:** this design's pull request.

### D18. One dependency root per language, and one version of each dependency

- **Decided:** 2026-11-25, by the monorepo maintainer role, on the coordinator's proposal.
- **Rule:** the repository has one `MODULE.bazel`, one uv workspace, one `go.mod`, one Cargo workspace, one pnpm workspace, and one root `Package.swift`, each with one lock file that Bazel reads. There's no `go.work`, because no written case needs one. A second version of a dependency needs a dated entry in the exceptions file ([appendix H](H-dependencies-and-renovate.md)).
- **Undo:** split a root by pull request. Each split brings back one more lock file to update.
- **Record:** this design's pull request.

### D17. Standard hosted runners only, and no paid build service

- **Decided:** 2026-11-25, by the monorepo maintainer role, inside the operator's rule that no paid service starts without approval.
- **Rule:** the public repository's CI runs on GitHub's standard hosted runners, Linux for most jobs and macOS for Swift builds. It uses no larger runner, no paid remote execution, and no self-hosted runner on `pull_request` events. The companion keeps the private runner label.
- **Undo:** change the runner labels, after the operator approves any cost.
- **Record:** this design's pull request.

### D16. `bazel-diff` computes affected targets

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** CI hashes the build graph per commit with `bazel-diff` and compares the base's hashes with the head's. The hashes for a `main` commit are saved once and reused. The other candidate, `target-determinator`, configures both revisions on every run ([appendix B](B-alternatives.md#target-determinator)).
- **Undo:** swap the plan job's tool. The test jobs take a target list from either.
- **Record:** this design's pull request.

### D15. The rules stay nested Bazel modules

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** the 13 rules modules keep their own `MODULE.bazel` files under `factory/build/rules`, and the root loads them through `local_path_override`. Labels such as `@tbhb_rules//lint:defs.bzl` don't change. A standalone repository can load them through `git_override` with a `strip_prefix` at its own cutover. A test fails when a nested module declares a version that differs from the root's.
- **Undo:** fold them into the root module, which rewrites every load statement and cuts off standalone consumers.
- **Record:** this design's pull request.

### D14. The layout goes by role, then project, with three dependency rules

- **Decided:** 2026-11-25, by the operator, from the coordinator's sketch. The monorepo maintainer role refined it in [appendix C](C-layout-and-ownership.md).
- **Rule:** top-level folders are `factory/`, `products/`, `lib/`, `docs/`, and `third_party/`. Products depend on `lib/` and published modules only, the factory never depends on a product, and Bazel visibility enforces both. Each project root's `OWNERS` file names its owning role, and the companion mirrors the public paths. The command-line tool's folder is open in D23.
- **Undo:** move folders by pull request. The rules live in build files and one test.
- **Record:** the operator's brief of 2026-11-25.

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

- **Decided:** 2026-11-25, by the operator.
- **Rule:** tools, agent-config, agent-memories, ops, wiki, research, spikes, event-gateway, rules, dotfiles, evals, dashboard, design, agent-capacity, runner-smoke, pm, sm, renovate-config, telemetry, and work move. afakt, avirt, afenc, and ascm stay standalone and are consumed, if at all, as Bazel modules at pinned versions. The layout leaves room for products and names none.
- **Undo:** import or extract a repository later.
- **Record:** the operator's brief of 2026-11-25.

### D10. One fresh public monorepo, one thin private companion, and no imported history

- **Decided:** 2026-11-25, by the operator. The import rule is the operator's later ruling of the same day, which replaced the brief's first wording.
- **Rule:** the factory moves into one new public repository, and private values go to one new private companion. Each repository comes in as one reorganizing copy commit that puts its files where the layout wants them and names the source repository and commit. No history comes into the monorepo for any repository, public or private, and the archived originals keep it.
- **Undo:** one-way door for each published commit. The move itself can stop after any import, because old repositories are archived and not deleted.
- **Record:** the operator's brief of 2026-11-25, addendum 5.

## Open items at a glance

| Item | Question | Recommendation |
| --- | --- | --- |
| D1 | Four factory-only repositories join, and awrel, gomodscan, and the Ansible collections stay out | yes |
| D2 | Each agent App reaches every path | yes |
| D3 | Runs on a host share one Git object store | yes |
| D4 | The operator signs off each private-origin import | yes |
| D6 | The private-name list is an Actions secret in the public repository | yes |
| D7 | Work on the in-house merge queue stops | yes |
| D8 | The tier and worktree records change to fit one repository | yes |
| D9 | Import pull requests skip the size limit | yes |
| D23 | The tool lives at `factory/tbhbdev` | yes |
| D24 | The organization is renamed as the last step | yes, after the rehearsal |
| D25 | The imports role runs the relevance pass by appendix I's criteria | yes |
| D26 | The private Project holds issues from both repositories | yes |
