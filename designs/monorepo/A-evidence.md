---
type: Design Appendix
title: One public monorepo for the factory evidence
description: What the repository reads, counts, docs, and the brief show for the monorepo design, each labeled proven, existing, new, or unverified.
tags: [monorepo, evidence, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo bzlmod xdist gomodscan promisor kexts evals avirt afenc hardlinks inodes determinator -->

# One public monorepo for the factory evidence

Back to [the design](README.md).

Labels: **proven** means a read or count run on 2026-11-25 against each repository's `origin/main` in the clones on one agent host, **existing** means a file or record at the named place, **per docs** means the vendor's documentation as read on 2026-11-25, **new** means this design proposes it, and **unverified** means nobody checked it. "The brief says" marks facts from the task brief that this pass didn't re-read.

The clones were fetched at different times, and the rules clone's `origin/main` dated from 2026-11-23 for the counts. A later fetch on 2026-11-25 read commit f0fdf1d for the tier 2 table. This pass had no access to the GitHub API, so it read no live ruleset, App installation, workflow run, or billing figure.

## Change counts

Each row counts first-parent commits on `origin/main` from 2026-10-28 to 2026-11-25 (proven). "Dependency" counts subjects that start with `chore(deps)` or `fix(deps)`. "First-party pin" counts dependency subjects that name the rules, the Renovate presets, the tools, or the grouped `pins` update.

| Repository | Landed | Dependency | First-party pin |
| --- | --- | --- | --- |
| ops | 363 | 51 | 19 |
| rules | 306 | 119 | 15 |
| tools | 304 | 28 | 19 |
| agent-config | 124 | 11 | 7 |
| wiki | 112 | 36 | 26 |
| agent-memories | 76 | 17 | 13 |
| renovate-config | 73 | 27 | 23 |
| design | 58 | 34 | 23 |
| agent-capacity | 55 | 15 | 7 |
| spikes | 50 | 31 | 23 |
| research | 43 | 26 | 21 |
| dotfiles | 36 | 6 | 4 |
| evals | 28 | 16 | 13 |
| sm | 27 | 1 | 1 |
| telemetry | 15 | 1 | 0 |
| dashboard | 13 | 1 | 1 |
| runner-smoke | 11 | 5 | 1 |
| event-gateway | 8 | 0 | 0 |
| pm | 8 | 1 | 0 |
| `bazel-r2-cache` | 76 | 33 | 28 |
| github-metrics | 67 | 6 | 3 |
| go.tbhb.dev | 55 | 27 | 21 |
| Ansible collections | Unmeasured | Unmeasured | Unmeasured |
| work | Unmeasured | Unmeasured | Unmeasured |
| **In scope, 24 repositories (22 measured)** | **1,908** | **492** | **268** |

The measured repositories averaged 477 merges a week and 68 a day. Dependency updates account for about 26%, or 123 a week. The first-party pin count gives an upper bound of 67 a week because the grouped `pins` update includes third-party digests.

D1 brought the three newly counted repositories and the Ansible collections into scope. Counts for the work repository and the Ansible collections are unavailable. On 2026-11-25 the repositories were about three weeks old, within the 28-day window.

| Other repository | Landed | Dependency | First-party pin |
| --- | --- | --- | --- |
| awrel | 112 | 42 | 36 |
| ascm | 62 | 33 | 30 |
| afakt | 60 | 30 | 22 |
| avirt | 57 | 35 | 28 |
| gomodscan | 54 | 39 | 26 |
| afenc | 52 | 29 | 24 |

In 14 of the in-scope repositories, 200 landed dependency updates named a third-party package, and they had 144 distinct subjects (proven). The same update landed in more than one repository 56 times.

## Size and duplication

| Claim | Label | Source |
| --- | --- | --- |
| The 19 clones in the original size measurement hold about 5,100 tracked files and 23 MB, with about 30 MB of Git data. The brief gives about 94 MB packed for 23 repositories | proven, and the brief says | `git ls-files` and `git count-objects` per clone |
| Nearly every repository carries the same 16 or so root configuration files, such as `.bazelrc`, `MODULE.bazel`, `.gitleaks.toml`, `.vale.ini`, and `renovate.json5` | proven | root listings of 16 repositories |
| 16 in-scope repositories hold 61 workflow files, and ops, agent-config, and agent-memories hold 11 more | proven | `git ls-tree` on `.github/workflows` |
| Consumers pin the rules by full commit hash and sat on seven different rules commits | proven | each repository's `MODULE.bazel` and workflow pins |
| 24 repositories call the rules' reusable `repo-checks` workflow | proven | search of the clones |
| The Renovate caller runs hourly on weekdays and every two hours on weekends, about 144 runs a week for each repository | existing | the Renovate presets' reusable workflow |
| Tools has one build file, one `pytest_test` target over the whole tree with `timeout = "long"`, 206 Python files, about 105,000 lines, and no Go | proven | tools at `origin/main`, commit da15405 |
| A tools queue pass takes about 1,400 seconds | existing | the [merge queue design's evidence](../merge-queue/A-evidence.md), which cites its brief |
| afakt and awrel generate Go build files with gazelle 0.54.0 and `rules_go` 0.65.0, and CI checks them with `//:gazelle -mode=diff`. No repository uses gazelle's Python plugin | proven | their `MODULE.bazel`, `BUILD.bazel`, and the `repo-checks` workflow |

## Rules, review, and CI today

| Claim | Label | Source |
| --- | --- | --- |
| The size check fails a pull request that adds more than 400 code lines or 1,500 raw lines. A sizing review agent can override either limit for the exact head it reviewed. The rule already exempts a repository's first pull request and Renovate | existing | the rules repository's actions README |
| The review tiers are in the wiki's first decision record: tier 0 is CI only, tier 1 is one fresh run, and tier 2 is two | existing | `adrs/0001` |
| The rules repository's `AGENTS.md` has a "Tier 2 review paths" table of 50 rows, each a glob with class 2 or class 3 and a reason | existing | rules at `origin/main`, commit f0fdf1d, read on 2026-11-25 |
| This repository's main ruleset requires the `conclusion` check pinned to GitHub Actions and dismisses stale reviews on a push. It requires no approving review and no last-push approval, unlike the rules repository's ruleset in the row below | existing | `.github/rulesets/main.json` |
| No test time cap is written down. The nearest is Bazel's `medium` timeout of 300 seconds | proven | search of the rules clone and the wiki |
| The `pytest_test` macro enforces 100% branch coverage inside one target, and consumers can't change it | existing | the rules' Python module |
| The rules' main ruleset requires signatures, a pull request with one approval, merge commits only, and a `conclusion` check pinned to GitHub Actions. It has no bypass actors | existing | the checked-in ruleset file |
| Tools' CI runs on `merge_group`, reads a main-only `cache` environment, uses the shared remote cache, and runs `bazelisk test --config=ci //...` on a standard hosted runner | proven | tools' `ci.yml` at `origin/main` |
| Reusable workflows send private repositories to a self-hosted runner label, and public ones to hosted runners, because the private runner groups don't take public repositories | existing | the rules' reusable workflows |
| The remote cache has kinds of token for reading, for `main`, for pull requests in their own namespace, for trusted build hosts, and for end-to-end tests | existing | the remote cache repository's docs |
| The wiki's tier 0 workflow calls a reusable workflow in the rules repository, with path lists for auto-approval and for denial | existing | this repository's `.github/workflows/tier0-approval.yml` |
| The interim approval command posts its approvals as the reviews App, `tbhb-dev-agent-reviews`, and not as `tbhb` | existing, the brief says | the command's own docstring and its `APP_LOGIN` value, as the fifth fix brief gives them. This pass didn't read the script |
| No design for the Go port exists in this repository or in any clone on the host | proven | search of designs, branches, and clones |
| CI minutes and run counts per repository | unverified | no record found; plan step 1 measures them |

## Agent runs today

| Claim | Label | Source |
| --- | --- | --- |
| `tbhbdev alpha agent exec` keeps one bare mirror per host and repository, and copies a full clone from it for each run with `git clone --no-local --no-hardlinks`. It doesn't use `git worktree` | proven | tools' `alpha/agent/worktree.py` at `origin/main` |
| The copy is deliberate: the code says a copying transport avoids shared object inodes and readable alternates | existing | the same file |
| Cleanup is a manual command, and nothing prunes the mirror | proven | the same package |
| One agent host holds 297 run directories under the tools clone and 598 Bazel output bases, on a disk that is 76% full | proven | directory counts and `df` on that host |
| The launcher sets no output base and no disk cache, so each run clone gets its own output base | proven | the launcher and tools' `.bazelrc` |
| The brief describes the current state as a worktree per run from a host clone | the brief says | task brief; the code differs, as the first row shows |

## Vendor facts

| Claim | Label | Source |
| --- | --- | --- |
| The merge queue rule is available on public repositories under the Team plan and refused on private ones | existing | the merge queue design's evidence |
| Push rulesets, including file path restrictions, apply to private and internal repositories only | per docs | GitHub Docs, "About rulesets", push rulesets |
| Standard hosted runners are free on public repositories, larger runners always bill, and the Team plan includes 3,000 minutes a month for private repositories | per docs | GitHub Docs, "About billing for GitHub Actions" |
| A pull request's author can't approve their own pull request | per docs, not re-read in this pass | GitHub Docs, "About pull request reviews" |
| An issue transfers only between repositories of one owner, and never from a private repository to a public one | per docs | GitHub Docs, "Transferring an issue to another repository" |
| A reusable workflow must sit in `.github/workflows` at a repository's root | per docs | GitHub Docs, "Reuse workflows" |
| `git sparse-checkout` in cone mode always includes files at the root and keeps its settings per worktree | per docs | `git-sparse-checkout(1)`, "Internals: cone pattern set" |
| `git backfill` fetches missing blobs in batches and is marked experimental. The host has Git 2.54 | per docs, proven | `git-backfill(1)` and `git --version` |
| Bazel derives a workspace's output base from its path, and the repository cache is shared across workspaces | per docs | Bazel docs, "Output directory layout" and "External dependencies overview" |
| VFS for Git is in maintenance mode, never shipped for macOS, and its maintainers point to Scalar. Scalar ships with Git | per docs | the VFS for Git repository's README, and `scalar(1)` |
| `bazel-diff` hashes targets per revision and compares two hash files. `target-determinator` compares configured targets across two revisions | per docs | each project's README |
| A renamed organization keeps Git and web redirects for its repositories until the old login is reused, and its profile links, API calls by the old login, and team mentions don't redirect | per docs | GitHub Docs, "Renaming an organization" |
| A workflow that uses an action from a renamed repository fails, with no redirect | per docs | GitHub Docs, "Renaming a repository" |
| A workflow `paths` filter reads at most 300 changed files, and a workflow doesn't run when its match lies past them | per docs | GitHub Docs, "Workflow syntax for GitHub Actions", `on.pull_request.paths` |
| The merge queue's build concurrency and its merge limits are separate settings | per docs | GitHub Docs, "Managing a merge queue" |

## GitHub's repository limits

These come from GitHub Docs, "Repository limits". The brief gave most of them, and the sixth fix round re-read the page on 2026-11-25, which added the read rate and the object size (per docs).

| Limit | Kind | The monorepo |
| --- | --- | --- |
| 10 GB on disk | recommended | About 94 MB packed (the brief says), under 1% of it |
| 3,000 entries in a directory | recommended | The widest directory read is 148 entries, in tools (proven, four clones) |
| Depth of 50 | recommended | The deepest path read is 10 levels, and the layout adds at most 3 |
| 5,000 branches | recommended | One branch for each open pull request. Merged heads are deleted, and the launcher prunes the rest ([appendix E](E-agent-runs.md#cleanup)) |
| 6 pushes a minute | recommended | About 68 landings a day. At four pushes per landing, the average would be one push every five minutes. The push count remains unverified |
| 15 Git reads a second, counting fetches and clones | recommended | Far under it on average, and a burst is unverified ([Git reads](#git-reads)) |
| About 1 merge a minute | recommended | The queue merges in groups, at most one group for each finished pass ([appendix F](F-ci.md#the-merge-queue)) |
| 1,000 open pull requests on a base branch | recommended | Renovate is capped at 10 ([appendix H](H-dependencies-and-renovate.md#renovate)). The open count today wasn't read (unverified) |
| 1 MB in one object | recommended | No tracked file in the 19 counted clones passes it, and the largest is about 632 KB, a generated type file in rules (proven, `git ls-tree -r -l origin/main`). The `import` job's summary lists any file over 1 MB |
| 2 GB in one push, and 100 MB in one file | enforced | No import nears either. The repository's size check already fails a file over 20 MB |
| 300 files in a pull request's diff | enforced | Import pull requests pass it. Review and deploys don't depend on it ([appendix G](G-import-and-cutover.md#importing-one-repository)) |

### Git reads

Every CI job checks out the repository, so a workflow run makes one read for each job ([appendix F](F-ci.md#one-workflow)). That's about 8 for a small change and about 18 with 10 test shards, so five queue candidates that start together make about 90 reads in a burst (new, counted from the job table). The launcher adds one fetch for each run start on a host, plus one batched blob fetch for the run's cone ([appendix E](E-agent-runs.md)).

At 15 a second the recommended rate allows 900 reads a minute, and a day's total is far under it. A burst is the risk, and its real size is unverified. Three things bound it (new):

- **The launcher fetches one at a time for each host.** It fetches under the mirror's lock, and it skips the fetch when the mirror was fetched in the last minute.
- **A worktree fills its blobs in one batch.** It never fetches them file by file.
- **A run's reads spread out.** The `test`, `coverage`, and `records` jobs start only after `plan` finishes.

`tbhbdev` retries a throttled fetch with backoff, and so does each job's checkout step. Plan step 11 records the peak reads a minute after each import.

## What plan step 2 must prove

These six items are unverified, and the design depends on each:

1. The rules' own tests pass from the monorepo root as `@tbhb_rules` targets, or need a second Bazel invocation inside the nested workspace.
2. `--override_module` on the command line replaces a `local_path_override` in the root `MODULE.bazel`, so the companion can replace the stub.
3. `bazel-diff` gives correct affected targets under bzlmod with nested local modules, and its hashes survive a round trip through the remote cache.
4. Gazelle and Bazel behave in a cone-mode sparse worktree: no wrong build file edits, and a clear error for a missing package.
5. A worktree from a partial bare mirror fetches its missing blobs in one batch, and a reused worktree path gets a warm Bazel output base.
6. A Bazel pin and a workflow call still resolve against an archived repository.
