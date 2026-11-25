---
type: Design Appendix
title: One public monorepo for the factory alternatives
description: The options the monorepo design didn't take, with the reason for each and what would change the answer.
tags: [monorepo, alternatives, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo fsmonitor kexts Mononoke promisor determinator -->

# One public monorepo for the factory alternatives

Back to [the design](README.md).

## Ways to work in a large repository

The proposal uses three features of core Git: a partial clone that leaves out file contents until needed, a cone-mode sparse checkout, and worktrees from one mirror. This section compares it with the tools built for the same problem.

### Scalar

Scalar is a command that ships with Git. `scalar clone` makes a partial clone, turns on a cone-mode sparse checkout, and registers the clone for background maintenance (per docs). It's a thin layer of settings over the same Git features the proposal uses.

The launcher doesn't call it, for three reasons. Scalar manages one working tree per clone, and the launcher needs many worktrees on one object store. Scalar schedules its own background maintenance, and the launcher must run maintenance under its own mirror lock. Scalar also lays out its clone in a fixed shape that doesn't match a bare mirror.

The launcher copies the settings Scalar applies where they help, such as the file system monitor and the untracked cache. Scalar becomes the better choice if agents ever work in one long-lived clone per host instead of one worktree per run.

### Virtual filesystems

A virtual filesystem shows the whole tree and fetches a file when something opens it. Three exist, and none fits:

| Tool | State | Why not |
| --- | --- | --- |
| VFS for Git | Maintenance mode; Windows only in practice. The macOS port stopped when Apple deprecated kernel extensions (per docs) | Agent hosts run macOS and Linux. It also needs a server protocol that GitHub doesn't serve |
| Clients in the Cloud (CitC) at Google | Internal to Google, on its own version control system | Not available outside Google |
| EdenFS from the Sapling project | Open source, with a daemon per host and its own server side for full effect | A daemon and a mount per host is a new service to run on every agent host, for a tree of 5,100 files |

A virtual filesystem pays off when the checkout itself is too large to write to disk, or when nobody can say ahead of time which files a task needs. Neither holds here. The whole tree is 23 MB, and Bazel can name a target's dependencies before the checkout.

Bazel also reads every input of a build, so a lazy filesystem ends up fetching the same files a cone would have checked out. Agents add a second problem: a search across the whole tree makes a lazy filesystem fetch everything.

The answer changes if a cone for a typical run passes about a million files, or if cone computation takes longer than the checkout it saves. Plan step 13 records cone sizes, so the trend is visible.

### Partial copy clones

This is the fallback in D3. The host keeps a full mirror, and each run gets its own clone from it with `--filter=blob:none` and a sparse checkout. The run copies every commit and tree and only the file contents in its cone.

It keeps today's isolation: no run can write another run's objects or refs. It costs a copy of all history per run, which is small now and grows by about 1,700 commits every four weeks. It also leaves one Bazel output base per clone path unless the launcher pools paths the same way.

### Full copy clones, as today

The launcher's current behavior works at 23 MB. The copies aren't the problem today. The 297 leftover run directories and 598 output bases on one host are, and the cleanup in [appendix E](E-agent-runs.md#cleanup) fixes those under any of these options.

## Build structure

### One root module with no nested modules

Folding the rules into the root module makes every label a plain `//factory/rules/...` path and removes the nested workspaces. Standalone repositories then have to load the whole monorepo as one module to reach a rule, and they inherit every root dependency. The four open source tools stay standalone by decision, so the rules must stay loadable alone.

### Publishing the rules to a registry

A private or public Bazel registry gives standalone consumers versions instead of commit pins. It adds a release step for each rules change, and the rules changed 306 times in 28 days. It becomes worth it when outside users appear, and nothing in this design blocks it.

### A Go workspace file and a lock for each project

A `go.work` over one `go.mod` for each project keeps today's import paths, and a lock for each project lets one project upgrade alone. Both bring back the same update landing many times, which is the cost this move removes. The clean break rewrites import paths anyway, and [appendix H](H-dependencies-and-renovate.md#a-second-version) covers the project that truly needs its own version.

### Target determinator

`target-determinator` compares configured targets, so it catches a change in a build flag that a plain query misses. It checks out and configures both revisions on each run. `bazel-diff` writes one hash file per commit, which CI saves for each `main` commit and reuses, and it can also hash configured targets (per docs). The plan job's output is a target list either way, so the tool can change later.

### Path filters instead of a build graph

GitHub's `paths` filters need no tool and run per workflow. They can't see that a change in `lib/` affects a service that depends on it, so each filter list drifts from the real graph. GitHub also reads at most 300 changed files for a filter (per docs), so a large change can skip a job. No workflow in this design uses one.

## Repository shape

### Keep the repositories and fix the pins

Renovate could merge first-party pins faster, or repositories could follow `main` of the rules without a pin. Following `main` breaks reproducible builds, and faster pin merges still leave seven rules versions in use and a cross-repository change split over several pull requests. The pins are a symptom of the repository boundaries.

### Two monorepos, one public and one private

A full private monorepo beside the public one doubles the build setup and brings back pins between the two. The decided shape keeps the private side thin: values only, no build logic of its own.

### One private monorepo

Everything private removes the publication risk. It also loses GitHub's merge queue and free hosted runners, which the Team plan offers on public repositories only, and it hides work that's public today.

### Git submodules

Submodules keep each repository and pin it by commit in a parent. That's the pin problem again with a harder workflow, and agents handle submodules poorly.
