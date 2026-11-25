---
type: Design Appendix
title: One public monorepo for the factory import and cutover
description: The prerequisites, the import order, the steps for each repository, what moves and how, what the move removes with before and after numbers, and the organization rename.
tags: [monorepo, migration, renovate, bazel, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo tbhbdev gomodscan cutover evals OIDC -->

# One public monorepo for the factory import and cutover

Back to [the design](README.md).

## Prerequisites and their order

These land in the old repositories first. Each is useful even if the move stops.

| Order | Prerequisite | State today | Why first |
| --- | --- | --- | --- |
| 1 | The baseline measurements | None recorded | Without them the move can't show a gain |
| 2 | The spike on a scratch repository | Six unverified items in [appendix A](A-evidence.md#what-plan-step-2-must-prove) | Each one can change the design |
| 3 | Gazelle in every Go and Python repository | Go repositories use it and CI checks it. No repository uses the Python plugin, and tools has one hand-written build file | Affected-target CI needs one target per package, or every change affects everything |
| 4 | The Go port's layout, one package per directory | No design found in this pass; the brief says the port is under way | Gazelle writes one Go target per directory, so the port's layout decides the target graph |
| 5 | Tests inside a time cap | No cap is written down. Tools' one test target sets `timeout = "long"` | The proposal adopts Bazel's `medium` limit of 300 seconds per target, so a shard's time is predictable |
| 6 | The combined coverage rule | Coverage is checked inside one target | Per-package targets need a check across targets ([appendix F](F-ci.md#coverage)) |
| 7 | `bazel-diff` in one repository's CI | None | Proves the plan job where a mistake costs one repository |

Items 3 to 5 meet in tools. If the Go port replaces the Python tree soon, the port carries gazelle and the time cap, and nobody splits the Python target first. That call waits for the port's design.

One gate applies to the port and to nothing else here. A Go port pull request for the tool's front, its dispatch, or its plugin plumbing goes to the operator before it merges (D29). The scaffolding and the imports don't wait on that gate.

## Import order

| Wave | What lands | Why here |
| --- | --- | --- |
| 0 | The scaffolding of both repositories | The shared roots, the inline CI workflow, the rulesets, a folder, a `README.md`, and an `OWNERS` file for every planned project, the tier 2 path map, the area labels, and the two Projects exist before any code |
| 1 | renovate-config | Renovate arrives with it: the presets, the root `renovate.json5`, and the root Renovate workflow. Every later import's dependencies are then kept current from its first day |
| 2 | rules | Everything else loads them, and the lint gates arrive with them. After this wave an imported project reads the rules from the tree |
| 3 | tools | The import tool, the launcher, and the review tool live here, and the later waves use them from the monorepo |
| 4 | wiki, research, spikes | Prose, with tier 0 review and few build targets. They exercise the queue at volume with little risk |
| 5 | design, evals, pm, telemetry, dashboard, event-gateway, agent-capacity, work, dotfiles, runner-smoke, and any D1 additions | Independent projects, smallest first |
| 6 | agent-memories, sm, agent-config, ops | Private-origin, after the scan tooling has run on real hosts for some weeks. ops is last because it's the largest and holds the most private values |

The break is clean (D27). Neither new repository ever calls a workflow in an old repository, pins one, or extends a preset from one. The scaffolding's workflow therefore uses only inline steps and third-party actions pinned by commit hash, and renovate-config's lint targets wait one wave for the rules.

A repository that hasn't moved yet changes nothing. It keeps its pins on the rules and presets repositories at the commits it holds, and those pins stop moving once their targets are archived. Whether a pin and a workflow call still resolve against an archived repository is unverified, and plan step 2 tests it.

Between waves, plan step 11 measures queue wait and CI time. A wave doesn't start while the previous one's numbers are worse than the baseline.

## Importing one repository

1. **Sort the issues.** The relevance pass in [appendix I](I-issues-and-projects.md#the-relevance-pass) starts. It doesn't block the import, and it does block the archive.
2. **Announce and drain.** The run that prepares the import posts a freeze time in the old repository. Open pull requests merge or close before it, and agents get no new briefs for that repository.
3. **Freeze.** The Apps' write access to the old repository is removed, which takes the organization admin grant, and the import names the frozen commit.
4. **Map.** A path map lists every file at that commit as copied to a new path, dropped, or rewritten. Dropped files are the root configuration that the shared roots replace. Rewritten files are those that called, pinned, or imported an old repository.
5. **Import.** `tbhbdev alpha monorepo import <repository>@<commit>` builds one signed commit and opens a pull request. The commit holds the files at their new paths, the project's dependencies folded into the shared roots, its rows in the tier 2 path map, and the build files gazelle writes. Its message names the source repository and the full commit hash, and it has no parent in the old repository.
6. **Prove.** A required `import` job checks that each copied file is byte-identical to its source, that the map accounts for every source file, and that no executable reference to an old repository remains. The project's targets then build and test like any affected targets.
7. **Review.** GitHub's diff view shows at most 300 files (per docs), so the reviewer works from a local checkout of the exact head. The `import` job's summary lists the dropped, rewritten, and generated files, and the reviewer reads each of those (D9).
8. **Archive.** Once the repository has no open issue and no open pull request, its README gets one line that points to the new path, and it becomes an archive. `tbhbdev alpha monorepo archive` refuses while either count is above 0.

A private-origin repository adds the split, the scan, and the operator's sign-off from [appendix D](D-public-private.md#importing-a-private-origin-repository) between steps 4 and 5.

"Executable reference" means a workflow `uses:` line, a Bazel override or module dependency, a Renovate `extends` entry, a Go import path, or a clone address in a script. Prose that names an old repository as a dated record, such as a decision record or this design, stays as written. Agent instruction files are rewritten to the new paths in the import commit.

## What moves, and how

| Thing | How it moves |
| --- | --- |
| Open pull requests | Drained before the freeze. A straggler is made again against the new paths, as a new pull request |
| Issues | Through the relevance pass, then into the monorepo or the companion with one area label each ([appendix I](I-issues-and-projects.md)) |
| Releases and tags | Old ones stay on the archived repository, where they remain downloadable. New ones are tagged `<project>/vX.Y.Z` |
| Go import paths | The import rewrites each one to the root module's path, and the map lists the files it touched. The vanity import service keeps serving only the standalone modules |
| Bazel module consumers outside the monorepo | A standalone repository keeps its rules pins until its own cutover, which it schedules itself. It then points each override at the monorepo with `strip_prefix = "factory/rules/<module>"` |
| Renovate | One root `renovate.json5` and one root workflow, from wave 1 ([appendix H](H-dependencies-and-renovate.md#renovate)) |
| Tier 0 approval | One root workflow, written inline, with the path lists in [appendix C](C-layout-and-ownership.md#the-tier-0-workflow) |
| Reusable workflows | The monorepo calls none. The ones standalone repositories need are published from the root `.github/workflows` ([appendix F](F-ci.md#what-stays-for-standalone-repositories)) |
| Deploy workflows | Deploys that need private values move to the companion. Others run on each push to `main` and deploy when `bazel-diff` marks their target as affected, with no path filter |
| Scheduled jobs, such as nightly mutation tests | One root workflow each, with a matrix over projects |
| Trackers | Each tracker row that names a repository gets the new path. The wiki's own links keep working inside `docs/wiki` because they're relative |

## What this removes

| Measure | Before | After | Label |
| --- | --- | --- | --- |
| Pull requests that only move a first-party pin | Up to 54 a week across the 19 repositories | 0 inside the monorepo, plus the companion's one pin | proven count, new target |
| Third-party update pull requests | About 50 a week in 14 repositories, 36 of them distinct | One per lock file a day, plus one per major version | proven count, new target |
| Dependency updates as a share of landings | 25% | Under 10% | proven count, new target |
| Rules versions in use at once | 7 | 1 inside the monorepo | proven count |
| Copies of the root configuration files | About 16 files in each of 19 repositories | 16 | proven count |
| Workflow files | At least 72 | About 10 | proven count, new target |
| Renovate runs | About 144 a week for each repository, near 2,700 in all | About 144, plus the companion's | existing schedule |
| Pull requests for a rules change that needs a consumer fix | One in rules, then one for each consumer after its pin moves | 1 | existing process |
| Main rulesets and App installation lists to audit | One for each repository | 2 | existing |
| Bazel output bases on an agent host | 598 on one host, unbounded | At most the slot count | proven count |
| Leftover run directories | 297 for one repository on one host | 0 after the reaper runs | proven count |
| CI minutes a week, and median time to `conclusion` | Not measured | Recorded in plan steps 1 and 13 | unverified |

The repositories that stay standalone keep their pin updates. The four open source tools landed 104 first-party pin updates between them in the same 28 days (proven).

Plan step 1 fills the last row with one read of the Actions API for each repository: run count, billed minutes, and time from push to `conclusion`, over 28 days. Plan step 13 repeats it on the monorepo.

## What gets worse

- **A rules change runs everything.** Today it runs the rules' tests, and consumers find breakage later through a pin update. The monorepo finds it at once and pays for the full run each time.
- **One queue.** A slow or flaky test anywhere delays every landing. The flake handling that the merge queue design planned has no home here, and GitHub's queue only removes the failing entry.
- **One public log.** Ops and agent configuration work becomes visible as it happens.
- **A wider blast radius for root files.** A bad change to `MODULE.bazel` or a lock file breaks every project, where today it breaks one.
- **Clone-based tools.** Anything that assumes one repository for each project, such as a tracker keyed by repository, needs a project field instead.
- **No history in one place.** `git log` and `git blame` in the monorepo stop at the import commit, and the earlier history is in the archive that the commit names.

## The organization rename

If D24 is yes, renaming `tbhb-dev` to `tbhbdev` is the last migration step, after the last archive and the after numbers. The clean break is what makes it small: by then the monorepo's own files hold no workflow call or pin that spells the organization.

| Reference class | Behavior after a rename | Label | Fix |
| --- | --- | --- | --- |
| Git remotes in host mirrors, clones, and the companion's pin, and web links to repositories, issues, and pull requests | Redirected, until someone creates a repository of the same name under the old login | per docs | The launcher rewrites remotes on its next fetch |
| Links to the organization's profile and its Projects | Not redirected | per docs for the profile, unverified for Projects | Edit the links in prose |
| Actions `uses:` lines in standalone repositories that call the monorepo | Not redirected for a renamed repository, and unverified for a renamed organization | per docs, unverified | Edit each caller in the same hour |
| REST and GraphQL calls that spell the old login | Fail for the organization's own endpoints | per docs | One setting in `tbhbdev` holds the login |
| Bazel `git_override` remotes and Renovate `github>` presets in standalone repositories | Follow Git's redirect | unverified | Edit at each repository's next update |
| App installations and their permissions | Kept, because they key on the organization's ID | unverified | None expected |
| Webhook payloads and anything keyed on `owner/repo`, such as event gateway routes and allow lists | Carry the new name from the rename on | unverified | Key on repository ID, or edit in the same change |
| OpenID Connect (OIDC) subject claims in cloud trust policies, such as the remote cache's | Carry the new name | unverified | Trust the owner ID claim, or edit the policy first |
| Go module paths | Unchanged, because they use the vanity domain | existing | Point the vanity service at the new name |
| Package and container namespaces, badges, and prose | Not redirected | unverified | A search of both repositories for the old login must return nothing |

The step runs in this order:

1. **Rehearse.** A run under the organization admin grant creates a free throwaway organization with one repository, one action, one App installation, and one webhook, then renames it. The run records what each class above did.
2. **Prepare.** Every "edit" fix above is ready as a pull request, and settings that can key on an ID already do.
3. **Rename.** A run under the same grant renames the organization and at once creates a free organization on the old login, with no repositories, so nobody else can claim it. An empty placeholder is expected to leave the redirects in place, and the rehearsal confirms that first.
4. **Check.** A clone by the old address works, a standalone repository's CI passes, an agent run starts, a webhook reaches the event gateway, and a deploy gets its cloud token.

## Stopping partway

The move can stop after any wave. Imported repositories are archived, not deleted, and the ones not yet imported keep working on the pins they already hold. Those pins no longer move, so a long stop means reopening the rules archive or finishing the imports. Reversing an import of a public repository means taking it out of the archive and removing the folder. Reversing an import of a private-origin repository can't unpublish what was pushed.
