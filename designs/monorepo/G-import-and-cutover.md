---
type: Design Appendix
title: One public monorepo for the factory import and cutover
description: The import order, the steps for each repository, how pull requests, issues, releases, and automation move, what the move removes with before and after numbers, and the prerequisites.
tags: [monorepo, migration, renovate, bazel, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo tbhbdev xdist gomodscan cutover evals -->

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

Items 3 to 5 meet in tools. If the Go port replaces the Python tree soon, the port carries gazelle and the time cap, and nobody splits the Python target first. The tools maintainer makes that call when the port's design exists.

## Import order

| Wave | Repositories | Why here |
| --- | --- | --- |
| 1 | rules, renovate-config | Everything else loads them. After this wave an imported repository reads the rules from the tree, with no pin |
| 2 | tools | The import tool, the launcher, and the review tool live here, and the later waves use them from the monorepo |
| 3 | wiki, research, spikes | Prose, with tier 0 review and few build targets. They exercise the queue at volume with little risk |
| 4 | design, evals, pm, telemetry, dashboard, event-gateway, agent-capacity, work, and any D1 additions | Independent projects, smallest first |
| 5 | dotfiles, runner-smoke | They wait on D5 |
| 6 | agent-memories, sm, agent-config, ops | Private-origin, by squash, after the scan tooling has run on real hosts for some weeks. ops is last because it's the largest and holds the most private values |

Between waves, plan step 9 measures queue wait and CI time. A wave doesn't start while the previous one's numbers are worse than the baseline.

## Importing one public repository

1. **Announce and drain.** The owning role posts a freeze time. Open pull requests merge or close before it. Agents get no new briefs for that repository.
2. **Freeze.** The owning role makes the old repository read-only for agents by removing the Apps' write access.
3. **Import.** `tbhbdev alpha monorepo import <repository> <path>` runs `git subtree add --prefix=<path> <URL> main` on a branch and opens a pull request. The old history arrives unchanged, joined by one signed merge commit.
4. **Prove.** A required check confirms that the tree at `<path>` equals the old repository's tree at the named commit. Nested workflow files under `<path>/.github` do nothing there.
5. **Adapt.** Follow-up pull requests of normal size delete the copied root configuration, fold the project's dependencies into the root manifests, add `OWNERS`, run gazelle, and move any workflow that still matters to the root.
6. **Move the rest.** Issues transfer, and the items in the next section follow.
7. **Archive.** The old repository's README gets one line that points to the new path, and the repository becomes an archive.

The project doesn't build between steps 3 and 5, so the import pull request's own paths are outside the affected set until the first adapt pull request adds them.

## What moves, and how

| Thing | How it moves |
| --- | --- |
| Open pull requests | Drained before the freeze. A straggler is reapplied with `git am --directory=<path>` from its patches, by the agent that owns it, as a new pull request |
| Issues | `gh issue transfer` to the monorepo, which keeps comments. The import tool creates an `area/<project>` label first and applies it. Issues from private repositories follow [appendix D](D-public-private.md#importing-a-private-origin-repository) |
| Releases and tags | Old ones stay on the archived repository, where they remain downloadable. New ones are tagged `<path>/vX.Y.Z`, the form Go expects for a module in a subdirectory |
| Go import paths | Each project keeps its `go.mod` and module path. The vanity import service points the path at the monorepo and the subdirectory. That needs a recent Go toolchain on the installer's side (unverified), which is one reason D1 leaves awrel out |
| Bazel module consumers | Standalone repositories change each rules override to the monorepo's URL with `strip_prefix = "factory/build/rules/<module>"` |
| Renovate | One root `renovate.json5` replaces the per-repository files and callers. The presets stay published from `factory/build/renovate`, and standalone repositories extend them by that subpath (per docs, unverified here) |
| Tier 0 approval | One root workflow, with the path lists in [appendix C](C-layout-and-ownership.md#review-tiers-by-path). The wiki's caller workflow goes away with its repository |
| Reusable workflows | They move to the root `.github/workflows` and stay callable by standalone repositories |
| Deploy workflows | Deploys that need private values move to the companion. Others move to the root with a path filter |
| Scheduled jobs, such as nightly mutation tests | One root workflow each, with a matrix over projects |
| Agent briefs, memories, and docs that name a repository | The import tool prints every `tbhb-dev/<name>` reference it finds, and the adapt pull requests fix them |
| Trackers | Each tracker row that names a repository gets the new path. The wiki's own links keep working inside `docs/wiki` because they're relative |

## What this removes

| Measure | Before | After | Label |
| --- | --- | --- | --- |
| Pull requests that only move a first-party pin | Up to 54 a week across the 19 repositories | 0 inside the monorepo, plus the companion's one pin | proven count, new target |
| Third-party update pull requests | About 50 a week in 14 repositories, 36 of them distinct | One per distinct update | proven count, new target |
| Dependency updates as a share of landings | 25% | Under 10% | proven count, new target |
| Rules versions in use at once | 7 | 1 inside the monorepo | proven count |
| Copies of the root configuration files | About 16 files in each of 19 repositories | 16 | proven count |
| Workflow files | At least 72 | About 10 | proven count, new target |
| Renovate runs | About 144 a week for each repository, near 2,700 in all | About 144, plus the companion's | existing schedule |
| Pull requests for a rules change that needs a consumer fix | One in rules, then one for each consumer after its pin moves | 1 | existing process |
| Main rulesets and App installation lists to audit | One for each repository | 2 | existing |
| Bazel output bases on an agent host | 598 on one host, unbounded | At most the slot count | proven count |
| Leftover run directories | 297 for one repository on one host | 0 after the reaper runs | proven count |
| CI minutes a week, and median time to `conclusion` | Not measured | Recorded in plan steps 1 and 11 | unverified |

The repositories that stay standalone keep their pin updates. The four open source tools landed 104 first-party pin updates between them in the same 28 days (proven).

Plan step 1 fills the last row with one read of the Actions API for each repository: run count, billed minutes, and time from push to `conclusion`, over 28 days. Plan step 11 repeats it on the monorepo.

## What gets worse

- **A rules change runs everything.** Today it runs the rules' tests, and consumers find breakage later through a pin update. The monorepo finds it at once and pays for the full run each time.
- **One queue.** A slow or flaky test anywhere delays every role's landings. The flake handling that the merge queue design planned has no home here, and GitHub's queue only removes the failing entry.
- **One public log.** Ops and agent configuration work becomes visible as it happens.
- **A wider blast radius for root files.** A bad change to `MODULE.bazel` or a lock file breaks every project, where today it breaks one.
- **Clone-based tools.** Anything that assumes one repository for each project, such as a tracker keyed by repository, needs a project field instead.

## Stopping partway

The move can stop after any wave. Imported repositories are archived, not deleted, and the ones not yet imported keep working with their pins pointed at the monorepo's rules. Reversing an import of a public repository means taking it out of the archive and removing the folder. Reversing an import of a private-origin repository can't unpublish what was pushed.
