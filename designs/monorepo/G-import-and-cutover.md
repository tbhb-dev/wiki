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

Measure the old repositories and prove the CI design in a scratch repository before scaffolding.

| Order | Prerequisite | State today | Why first |
| --- | --- | --- | --- |
| 1 | The baseline measurements | None recorded | Without them the move can't show a gain |
| 2 | The spike on a scratch repository | Six unverified items in [appendix A](A-evidence.md#what-plan-step-2-must-prove) | Each one can change the design |
| 3 | Combined coverage and `bazel-diff` in scratch CI | Coverage is checked inside one target, and the source repositories lack the plan job | Per-package targets need a combined coverage check, and affected-target CI needs a tested plan job ([appendix F](F-ci.md#coverage)) |

Go ports follow import (D41). A Go port pull request for the command-line tool's front, its dispatch, or its plugin plumbing goes to the operator before it merges (D29).

## Import order

| Wave | What lands | Why here |
| --- | --- | --- |
| 0 | The scaffolding of both repositories | The shared roots, the inline CI workflow, the rulesets, a folder, a `README.md`, and an `OWNERS` file for every planned project, the tier 2 path map, the area labels, and the two Projects exist before any code |
| 1 | renovate-config | Its presets, pin script, and tests go to `lib/renovate-config`, and its three workflows go to `.github/workflows/`. The root `renovate.json5` uses the presets from the first day (D40) |
| 2 | rules | Everything else loads them, and the lint gates arrive with them except for the review-policy checker test (D42). After this wave an imported project reads the rules from the tree |
| 3 | tools into `factory/cli` and ops into `ops/`, in parallel | The CLI import adds the review-policy checker test (D42), and later waves use the tool from the monorepo. The ops import puts its private values into the companion's `ops/` and passes the private-origin gate before publication |
| 4 | wiki, research, spikes | Prose, with tier 0 review and few build targets. They exercise the queue at volume with little risk |
| 5 | design, evals, pm, telemetry, dashboard, event-gateway, agent-capacity, work, dotfiles, runner-smoke, github-metrics, bazel-r2-cache, go.tbhb.dev, and the Ansible collections | Independent imports, smallest first. Runner-smoke adds four root workflows and six sample projects under `ops/runner-smoke/`. The Ansible collections land at `ops/ansible/collections/` |
| 6 | agent-memories, sm, agent-config | The remaining private-origin repositories, after the scan tooling has run on real hosts for some weeks |

The break is clean (D27). Neither new repository ever calls a workflow in an old repository, pins one, or extends a preset from one. The scaffolding's workflow therefore uses only inline steps and third-party actions pinned by commit hash, and renovate-config's lint targets wait one wave for the rules.

The 2026-11-26 coordination call (D42) puts the test with the command-line tool because it exercises the tool's checker. In wave 2, the rules import's path map marks `//lint:review_policy_test` as a drop and gives that reason. The import rewrites the affected build files to omit the test and removes the tools pin. In wave 3, the CLI import adds the test under `factory/cli`, with a dependency on the policy data in `lib/rules`. The rules import's pull request body names the gap in monorepo coverage from the rules import until the CLI import merges. To undo this decision, move the test back to `lib/rules` and allow that one dependency on `factory/cli`.

A repository that hasn't moved yet changes nothing. It keeps its pins on the rules and presets repositories at the commits it holds, and those pins stop moving once their targets are archived. Whether a pin and a workflow call still resolve against an archived repository is unverified, and plan step 2 tests it.

Between waves, plan step 11 measures queue wait and CI time. A wave doesn't start while the previous one's numbers are worse than the baseline.

## Importing one repository

1. **Sort the issues.** The relevance pass in [appendix I](I-issues-and-projects.md#the-relevance-pass) starts with a `tbhb-agent` sorting run. A fresh `tbhb-agent` run on another provider confirms each close list before any issue on it closes (D34). It doesn't block the import, and it does block the archive.
2. **Announce and drain.** `tbhb-agent` posts a freeze time in the old repository. Open pull requests merge or close before it, and agents get no new briefs for that repository.
3. **Freeze.** `tbhb-agent` removes the Apps' write access to the old repository, under the organization admin grant, and the import names the frozen commit.
4. **Map.** A path map lists every file at that commit as copied to a new path, dropped, or rewritten. It records the configuration that the shared roots replace, the layout-specific drops in [appendix C](C-layout-and-ownership.md#where-each-repository-goes), and rewrites needed to use the new paths and dependency roots.
5. **Import.** `tbhbdev alpha monorepo import <repository>@<commit>` builds one signed commit and opens a pull request. It imports the project as it is, with the mapped layout changes, shared dependency entries, and tier 2 path map rows needed to land it. The source language and target granularity carry over. The message names the source repository and full commit hash. The commit has no parent in the old repository.
6. **Prove.** The required `import` job fetches a public source at the named commit and runs `verify` against it. The check compares every copied file byte for byte, checks that the map accounts for every source file, and rejects executable references to old repositories. [Appendix D](D-public-private.md#what-public-ci-proves-for-a-private-origin-import) describes the checks for private sources. The project's targets build and test like any affected targets.
7. **Review.** GitHub’s diff view shows at most 300 files (per docs). The reviewer uses a local checkout of the exact head to read the path map and every dropped, rewritten, or generated file. If the import exceeds either size limit, a sizing review agent uses the passing byte-identical import check for that head as evidence for an override under the existing size rule (D9). For a private-origin import, the companion's comparison supplies that evidence.
8. **Split the build.** Right after a project's import, land a separate change that splits its build into fine-grained targets. Use Gazelle where a Gazelle extension exists. Keep the split out of the import commit so `verify` can compare the import with its source (D41). The split keeps test targets within Bazel's `medium` timeout. Go ports happen after import.
9. **Archive.** Once the repository has no open issue and no open pull request, its README gets one line that points to the new path, and it becomes an archive. `tbhbdev alpha monorepo archive` refuses while either count is above 0.

For a private-origin repository, step 5 builds the signed copy commit in a private workspace. The split, scans, and two independent reviews in [appendix D](D-public-private.md#importing-a-private-origin-repository) clear that exact commit before the import tool pushes it and opens the public pull request.

"Executable reference" means a workflow `uses:` line, a Bazel override or module dependency, a Renovate `extends` entry, a Go import path, or a clone address in a script. Prose that names an old repository as a dated record, such as a decision record or this design, stays as written. Agent instruction files are rewritten to the new paths in the import commit.

## What moves, and how

| Thing | How it moves |
| --- | --- |
| Open pull requests | Drained before the freeze. A straggler is made again against the new paths, as a new pull request |
| Issues | Through the relevance pass, then into the monorepo or the companion with one area label each ([appendix I](I-issues-and-projects.md)) |
| Releases and tags | Old ones stay on the archived repository, where they remain downloadable. New ones are tagged `<project>/vX.Y.Z` |
| Go import paths | The import rewrites each one to the root module's path, and the map lists the files it touched. The vanity import service keeps serving only the standalone modules |
| Bazel module consumers outside the monorepo | A standalone repository keeps its rules pins until its own cutover, which it schedules itself. It then points each override at the monorepo with `strip_prefix = "lib/rules/<module>"` |
| Renovate | One root `renovate.json5` and three workflows in `.github/workflows/`, from wave 1 ([appendix H](H-dependencies-and-renovate.md#renovate)) |
| Tier 0 approval | One root workflow, written inline, with the path lists in [appendix C](C-layout-and-ownership.md#the-tier-0-workflow) |
| Reusable workflows | The monorepo calls none. The ones standalone repositories need are published from the root `.github/workflows` ([appendix F](F-ci.md#what-stays-for-standalone-repositories)) |
| Deploy workflows | Deploys that need private values move to the companion. Ops deploys that need private values read them from the companion's `ops/`. Under the recommendation to keep `ops/` outside Bazel, its workflows use the deployment tools directly. Public Bazel deploys run on a push to `main` when `bazel-diff` marks their target as affected, with no path filter |
| Scheduled jobs, such as nightly mutation tests | One root workflow each, with a matrix over projects |
| Trackers | Each tracker row that names a repository gets the new path. The wiki's own links keep working inside `docs/wiki` because they're relative |

## What this removes

| Measure | Before | After | Label |
| --- | --- | --- | --- |
| Pull requests dedicated to first-party pin updates | Up to 67 a week across the 22 measured repositories | 0 inside the monorepo, plus the companion's one pin | proven count, new target |
| Third-party update pull requests | About 50 a week in 14 repositories, 36 of them distinct | One per lock file a day, plus one per major version | proven count, new target |
| Dependency updates as a share of landings | About 26% | Under 10% | proven count, new target |
| Rules versions in use at once | 7 | 1 inside the monorepo | proven count |
| Copies of the root configuration files | About 16 files in each of the 16 repositories in the root listing | 16 | proven count |
| Workflow files | At least 72 | About 10 | proven count, new target |
| Renovate runs | About 144 a week for each repository, near 3,500 across the 24 in-scope repositories | About 144, plus the companion's | existing schedule |
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

Renaming `tbhb-dev` to `tbhbdev` is the last migration step, after the last archive and the after numbers (D24). The rename proceeds only after the rehearsal passes. The clean break is what makes it small: by then the monorepo's own files hold no workflow call or pin that spells the organization.

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
| Team mentions, written `@<organization>/<team>`, in issues, pull requests, and prose | Not redirected | per docs, and whether the organization has a team is unverified | Edit the ones in files before the rename, and mention again in open issues after it |
| Go module paths | The monorepo uses `github.com/tbhb-dev/tbhbdev`. Standalone modules on the vanity domain keep their paths | design | Update the monorepo's `go.mod` and imports at the rename if it still uses the GitHub path. Point the vanity service at the new name for standalone modules |
| Package and container namespaces, badges, and prose | Not redirected | unverified | A search of both repositories for the old login must return nothing |

The rename follows this order.

1. **Rehearse.** `tbhb-agent`, under the organization admin grant, creates a free throwaway organization with one repository, one action, one App installation, one webhook, and one team that an issue mentions, then renames it. The rehearsal's record says what each class above did, team mentions among them.
2. **Prepare.** Every "edit" fix above is ready as a pull request, and settings that can key on an ID already do.
3. **Rename.** `tbhb-agent`, under the same grant, renames the organization and at once creates a free organization on the old login, with no repositories, so nobody else can claim it. An empty placeholder is expected to leave the redirects in place, and the rehearsal confirms that first.
4. **Check.** A clone by the old address works, a standalone repository's CI passes, an agent run starts, a webhook reaches the event gateway, and a deploy gets its cloud token. A search of both repositories and their open issues for a team mention under the old login returns nothing.

`tbhb` answers for the rename through D24, and `tbhb-agent` carries out each of the four steps.

## Stopping partway

The move can stop after any wave. Imported repositories are archived, not deleted, and the ones not yet imported keep working on the pins they already hold. Those pins no longer move, so a long stop means reopening the rules archive or finishing the imports. Reversing an import of a public repository means taking it out of the archive and removing the folder. Reversing an import of a private-origin repository can't unpublish what was pushed.
