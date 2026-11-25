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

<!-- cspell:words monorepo bzlmod gomodscan determinator evals avirt afenc -->

# One public monorepo for the factory decisions

Back to [the design](README.md).

## Open

### D1. Do github-metrics, graph, bazel-r2-cache, and go.tbhb.dev join, while awrel, gomodscan, and the Ansible collections stay standalone?

- **Recommendation:** yes.
- **Why:** the first four serve only the factory, so they gain from one build and lose nothing. awrel, gomodscan, and the Ansible collections are tools that other people install by a public path and a version, like the four standalone open source tools. Nobody read graph in this pass, so its place rests on the coordinator's description.
- **If yes:** the four join in plan step 9 at the paths in [appendix C](C-layout-and-ownership.md#where-each-repository-goes). awrel keeps its own repository, releases, and pins.
- **If no:** name the repositories to flip. Bringing awrel in costs a vanity import path that points at a subdirectory, which needs Go 1.25 or later on every installer's machine (unverified), and tags of the form `<path>/vX.Y.Z`.
- **Reversible:** yes. A repository can join later by the same import, and a folder can leave by `git subtree split`.

### D2. May each agent App reach every path of the monorepo?

- **Recommendation:** yes.
- **Why:** a GitHub App's permissions apply to a whole repository (per docs). Today an App installed on only some repositories can't touch the others, and one repository removes that limit. Push rules that restrict file paths don't exist for public repositories (per docs).
- **If yes:** path limits rest on the main ruleset, on review by the owning role, and on tier 2 for the paths in the root policy. An agent can still push a branch that touches any path, and it can't merge one without those.
- **If no:** sensitive folders, such as agent configuration and ops, stay in their own repositories, and the pins between them stay.
- **Reversible:** yes, by moving a folder back out with `git subtree split`.

### D3. May agent runs on one host share one writable Git object store?

- **Recommendation:** yes.
- **Why:** the launcher copies a full clone per run today, on purpose, so runs share no objects. Worktrees from one mirror share objects and refs, which makes a run cost its sparse files instead of the whole history. The monorepo's objects are all public, so the exposure is to a run that changes another run's refs, not to a run that reads them.
- **If yes:** reviews check out a commit hash fetched from GitHub and never a local branch name, the launcher alone prunes and collects garbage, and an extended reference-transaction hook refuses updates to another run's branch. The companion keeps copy clones, because its objects are private.
- **If no:** each run gets a partial, sparse copy clone from a full host mirror. That keeps today's isolation and the sparse view, and each run copies every commit and tree, which grows with history ([appendix B](B-alternatives.md#partial-copy-clones)).
- **Reversible:** yes. The launcher keeps both modes behind one setting.

### D4. Does the operator sign off each private-origin import before it's pushed?

- **Recommendation:** yes, once each for ops, agent-config, agent-memories, and sm.
- **Why:** publication is a one-way door. Each import is one squashed commit, so the operator reads a file list and a scan report instead of a history.
- **If yes:** the import tool writes the report on a host that holds the private-name list, and the push waits for the sign-off in the companion. The report lists every public file, every finding, and every file held back.
- **If no:** the ops maintainer and the agent-config maintainer sign off for their own repositories, and the operator learns of a leak after the fact.
- **Reversible:** no. A pushed commit counts as published even when it's deleted later.

### D5. May dotfiles and runner-smoke come in as one squashed commit each?

- **Recommendation:** yes.
- **Why:** each has exactly one commit with no signature (proven), and the main ruleset requires signatures. The rule for public repositories is to keep history, so this is an exception to it. The archived repositories keep the full history.
- **If yes:** both import by squash like a private repository, and each import commit names the source commit.
- **If no:** the admin role adds a ruleset bypass for those two imports, which loosens the signature rule on `main` for a moment and carries two unsigned commits forever.
- **Reversible:** no. History in the monorepo is never rewritten, so the choice at import holds.

### D6. May the public repository hold the private-name list as an Actions secret?

- **Recommendation:** yes.
- **Why:** host hooks are the first barrier, and a host with a broken hook has none. A CI job that scans each pull request and each queue candidate catches that case before `main`. The list itself is private, so the job needs it as a secret.
- **If yes:** the admin role stores the list as a repository secret. The scan job prints only a file and a line, never the match. Fork pull requests get no secret, so the queue candidate is their scan.
- **If no:** scanning stays on hosts only. A leak pushed from a host with a broken hook is found by a person or not at all.
- **Reversible:** yes. Delete the secret and the job.

### D7. Does work on the in-house merge queue stop now?

- **Recommendation:** yes, and its design is marked superseded once plan step 10 lands.
- **Why:** that queue exists for private repositories that GitHub's queue refuses. This move puts their code in a public repository and gives the companion an up-to-date check instead of a queue. The operator's decisions D1, D2, D3, and D17 in that design then cover nothing.
- **If yes:** ops and agent-config merge without a queue until their imports, as they do today. Nobody creates the queue's App or its Temporal workflows.
- **If no:** the queue ships for ops and agent-config and serves them for the weeks until plan step 10.
- **Reversible:** yes. The design stays in the repository and can resume.

### D8. May the decision records for review tiers and for worktrees change to fit one repository?

- **Recommendation:** yes.
- **Why:** the tier record sorts changes partly by repository type, and one repository has no types. The worktree record says a clone's main worktree stays clean and runs get worktrees under it, and a bare mirror has no main worktree. Both are the operator's standing rules.
- **If yes:** two tier 2 pull requests amend them. Tiers key on paths alone, with the same three tiers and the same triggers. Worktrees live under the launcher's state directory.
- **If no:** the monorepo keeps a path-to-type table that copies today's repository types, and the launcher keeps a full clone beside the mirror only to satisfy the worktree record.
- **Reversible:** yes, by a later amendment.

### D9. May an import pull request skip the size limit and line-by-line review?

- **Recommendation:** yes, when the tree-equality check passes.
- **Why:** an import adds a whole repository, far past 1,500 lines, and the size check has no override. The content already passed review where it was written. A required check proves the tree under the target path equals the source repository's tree at a named commit, so nothing rides along.
- **If yes:** the size check exempts a pull request whose only change is one new prefix and whose equality check is green. One reviewer confirms the path and the source commit.
- **If no:** each import splits into pull requests under the limit. Tools alone has about 105,000 lines of Python, which is over 70 pull requests, and history can't survive the split.
- **Reversible:** yes. Remove the exemption after the last import.

## Decided

### D22. Old repositories are archived, never deleted

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** after its import, a repository gets a pointer in its README and becomes a read-only archive. Its issues, pull requests, releases, and tags stay readable there.
- **Undo:** take it out of the archive in repository settings.
- **Record:** this design's pull request.

### D21. No Scalar command and no virtual filesystem

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** the launcher uses partial clone, cone-mode sparse checkout, and `git maintenance` from core Git directly. It doesn't use the `scalar` command, VFS for Git, or a FUSE filesystem. [Appendix B](B-alternatives.md#scalar) gives the reasons and what would change them.
- **Undo:** adopt one later. Nothing in the layout depends on this.
- **Record:** this design's pull request.

### D20. The working name is `tbhb-dev/monorepo`

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** the design and the tools use this name until the admin role creates the repository in plan step 4. The operator can pick another name before then at no cost. The companion's name is set when it's created and never appears in the public repository.
- **Undo:** rename the repository. GitHub redirects the old name, and Go import paths use the vanity domain.
- **Record:** this design's pull request.

### D19. The design system goes under `lib/`, not `docs/`

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** the design repository is a workspace of published packages, so it lands at `lib/design`, where products may depend on it. The starting sketch put it under `docs/`.
- **Undo:** move the folder in one pull request.
- **Record:** this design's pull request.

### D18. One workspace file per language, and each Go project keeps its module path

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** a root `go.work` lists each Go project's `go.mod`, so import paths don't change and a product stays extractable. Python uses one uv workspace with one lock file, and JavaScript uses one pnpm workspace. Each language resolves one version of a third-party package for the whole repository.
- **Undo:** merge the `go.mod` files into one, which rewrites import paths.
- **Record:** this design's pull request.

### D17. Standard hosted runners only, and no paid build service

- **Decided:** 2026-11-25, by the monorepo maintainer role, inside the operator's rule that no paid service starts without approval.
- **Rule:** the public repository's CI runs on GitHub's standard hosted runners. It uses no larger runner, no paid remote execution, and no self-hosted runner on `pull_request` events. The companion keeps the private runner label.
- **Undo:** change the runner labels, after the operator approves any cost.
- **Record:** this design's pull request.

### D16. `bazel-diff` computes affected targets

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** CI hashes the build graph per commit with `bazel-diff` and compares the base's hashes with the head's. The hashes for a `main` commit are saved once and reused. The other candidate, `target-determinator`, configures both revisions on every run ([appendix B](B-alternatives.md#target-determinator)).
- **Undo:** swap the plan job's tool. The test jobs take a target list from either.
- **Record:** this design's pull request.

### D15. The rules stay nested Bazel modules

- **Decided:** 2026-11-25, by the monorepo maintainer role.
- **Rule:** the 13 rules modules keep their own `MODULE.bazel` files under `factory/build/rules`. The root loads them through `local_path_override`, and standalone repositories load them through `git_override` with a `strip_prefix`. Labels such as `@tbhb_rules//lint:defs.bzl` don't change anywhere.
- **Undo:** fold them into the root module, which rewrites every load statement and cuts off standalone consumers.
- **Record:** this design's pull request.

### D14. The layout goes by role, then project, with three dependency rules

- **Decided:** 2026-11-25, by the operator, from the coordinator's sketch. The monorepo maintainer role refined it in [appendix C](C-layout-and-ownership.md).
- **Rule:** top-level folders are `factory/`, `products/`, `lib/`, `docs/`, and `third_party/`. Products depend on `lib/` and published modules only, the factory never depends on a product, and Bazel visibility enforces both. Each folder's `OWNERS` file names its owning role, and the companion mirrors the public paths.
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
- **Undo:** import or split out a repository later.
- **Record:** the operator's brief of 2026-11-25.

### D10. One fresh public monorepo, one thin private companion, and no rewritten history

- **Decided:** 2026-11-25, by the operator.
- **Rule:** the factory moves into one new public repository, and private values go to one new private companion. Public repositories come in under their target path with history, through `git subtree add`. Private repositories come in as one squashed commit after a gitleaks scan, and nobody rewrites history at any point.
- **Undo:** one-way door for each published commit. The move itself can stop after any import, because old repositories are archived and not deleted.
- **Record:** the operator's brief of 2026-11-25.
