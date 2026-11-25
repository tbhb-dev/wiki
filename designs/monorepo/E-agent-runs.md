---
type: Design Appendix
title: One public monorepo for the factory agent runs
description: How agent runs and review runs work on the monorepo, with mirrors, worktrees, sparse cones, widening, disk use, cleanup, and Bazel caches.
tags: [monorepo, agents, git, worktree, sparse-checkout, bazel, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo buildfiles rdeps promisor tbhbdev fsmonitor refspec widenings cutover -->

# One public monorepo for the factory agent runs

Back to [the design](README.md).

The operator asked directly how agent runs work on a large repository. This appendix answers that. [Appendix B](B-alternatives.md#ways-to-work-in-a-large-repository) compares the answer with Scalar and with virtual filesystems.

## Where this starts

The brief describes today's runs as worktrees cut from a host clone. The launcher's code does something else: it keeps a bare mirror per host and copies a full clone from it for every run, so that runs share no objects (proven, [appendix A](A-evidence.md#agent-runs-today)). The proposal below is therefore a larger change than the brief implies, and D3 asks the operator about the part that shares objects.

The monorepo starts at about 5,100 files and 23 MB. At that size a full copy per run costs under a second and little disk. The proposal still matters now for two reasons. Cleanup is broken today, and a narrow view keeps an agent's searches and edits inside its task.

## The mirror

Each host keeps one bare mirror of the monorepo under the launcher's state directory. Only the launcher writes to it, and every write holds the mirror lock that exists today.

```sh
git clone --bare --filter=blob:none <monorepo URL> <state>/mirrors/tbhbdev.git
git -C <mirror> config remote.origin.fetch '+refs/heads/main:refs/remotes/origin/main'
git -C <mirror> fetch origin main          # at each run start, under the lock
```

`--filter=blob:none` makes a partial clone: the mirror has every commit and tree, and it fetches a file's contents the first time something needs them. A blob fetched for one run then serves every later run. The monorepo is public, so that fetch needs no credential.

The companion gets a plain full mirror and copy clones, as today. It's small, and its objects are private.

## A build run

1. **Fetch.** The launcher fetches `main` into the mirror, and skips the fetch when the mirror was fetched in the last minute.
2. **Compute the cone.** The launcher keeps one full checkout per host, the index worktree, which no agent uses. It moves that checkout to the fetched commit and asks Bazel which folders the run needs:

   ```sh
   bazel query 'buildfiles(deps(//factory/event-gateway/...))' --output=package
   ```

   `deps` lists everything the targets depend on, and `buildfiles` adds the build and rule files those packages load. The cone is the set of folders in the answer, plus the target paths themselves.
3. **Add the worktree.** The launcher takes a free slot path, described under [Bazel state](#bazel-state), and runs:

   ```sh
   git -C <mirror> worktree add --no-checkout -b agent/<name> <slot path> origin/main
   git -C <slot path> sparse-checkout set --cone <folders>
   git -C <slot path> checkout
   ```

   Cone mode always includes the files at the repository root, so `MODULE.bazel`, the lock files, and the lint configuration are present. The checkout fetches all missing blobs for the cone in one request (per docs, tested in plan step 2).
4. **Install hooks.** The launcher installs the same read-only hooks as today, plus the push hook from [appendix D](D-public-private.md#keeping-private-content-out).
5. **Run.** The agent builds and tests its own paths, such as `bazelisk test //factory/event-gateway/...`. It doesn't run `//...`, because a sparse tree doesn't hold every package. CI runs whatever else the change affects.

The run's brief names its target paths. A brief with no paths gets a full checkout, through the same `--full` flag an agent can ask for.

## Widening

An agent that needs a folder outside its cone runs one command:

```sh
tbhbdev alpha agent widen //lib/design/...     # a Bazel target pattern
tbhbdev alpha agent widen docs/wiki/designs    # or a path
```

The command computes the new target's dependencies in the index worktree, runs `git sparse-checkout add`, and appends the widening to the run record. It never narrows, so nothing an agent already saw disappears.

Three cases need care:

- **A search finds nothing.** A sparse tree hides files, so "no results" can be wrong. The agent's instructions for the monorepo say to search with `git grep <pattern> origin/main`, which reads the whole tree from the object store.
- **A change breaks a dependent.** Dependents aren't in the cone by default. `tbhbdev alpha agent widen --dependents` adds the direct ones, and CI tests all of them either way.
- **A file outside the cone.** Git refuses to add a path outside the cone unless told to, so an agent can't commit one by accident.

## A review run

A reviewer needs the same view the author had, and more. The review profile computes its cone from the pull request alone:

1. the folders of every changed file
2. the dependencies of every changed package, by the same query, at the head commit
3. the direct dependents of every changed package
4. the widenings in the author's run record, which the pull request body lists

Items 1 to 3 are a function of the base and head commits, so any reviewer on any host gets the same folders. The reviewer's worktree is detached at the exact head commit, fetched from GitHub by hash. A reviewer can widen freely, and a second review run starts from the same computed cone, not from the first reviewer's.

## Bazel state

Bazel keeps one output base per workspace path. Today each run clone has a new path, so each run starts cold and leaves an output base behind. One host holds 598 of them (proven).

- **Slot paths.** Worktrees live at a fixed set of paths, `<state>/worktrees/tbhbdev/slot-01` to `slot-NN`. A run takes a free slot under a lock and gives it back at the end. Bazel sees the same path again, so the next run reuses that slot's output base and its fetched external repositories. The count of output bases can't pass the count of slots.
- **A shared disk cache.** One `--disk_cache` folder per host serves every slot, so a target built in one slot isn't rebuilt in another. Bazel trims it to a size limit (per docs).
- **A shared repository cache.** Downloads of external dependencies are stored once per host, which is Bazel's default (per docs).
- **The remote cache.** Hosts read it with the trusted host token that exists today, and CI's results reach every host.

How much a warm slot saves is unverified, and plan step 2 measures it. A slot's output base holds results from an earlier run, which is the same trust the shared caches already need. Whether the launcher confines a run to its own folder today wasn't read in this pass (unverified).

## Disk budget per host

| Item | Bound | Today |
| --- | --- | --- |
| The mirror | All commits and trees, plus the blobs any run touched | about 30 MB of Git data for all in-scope repositories |
| The index worktree | One full checkout | about 23 MB |
| Run worktrees | Slots times cone size, at most 16 slots | 297 leftover directories for one repository |
| Output bases | One per slot, at most 16 | 598 on one host |
| Disk cache | A fixed limit, 50 GB to start | none shared |
| Repository cache | Grows with distinct dependency versions | shared by default |

The launcher refuses to start a run when the disk is over 90% full, and it reports the host in the dashboard at 80%. Slot count per host is a launcher setting, sized from the host's memory, because each active slot holds a Bazel server.

## Cleanup

Cleanup is a manual command today. It becomes the launcher's job, at each run start and from a daily timer on each host.

1. **At run end.** The launcher records the head commit, confirms the branch is on GitHub or the run made no commits, removes the worktree, and frees the slot. A run with commits that aren't on GitHub keeps its slot and is reported.
2. **Stale slots.** A slot idle for 24 hours whose run never ended cleanly goes through the same check and is freed.
3. **Local branches.** A branch in the mirror is deleted once its pull request is merged or closed, or after 14 days with no worktree.
4. **The mirror.** `git worktree prune` and `git maintenance run` go under the mirror lock, from the timer only. Agents' Git settings in a worktree turn automatic garbage collection off, as the mirror's do today.
5. **Bazel.** A slot unused for seven days gets `bazel clean --expunge`. The disk cache trims itself.
6. **The old layout.** After cutover, one command moves each old run clone to trash after the same branch check that manual cleanup uses today.

## What sharing changes

Worktrees write their objects and their branch refs into the mirror. A run can therefore damage another run's branch, or delete objects another run needs, by running the wrong Git command. Three things limit that:

- Git objects are named by the hash of their contents, so a run can't replace an object under the same name.
- Review runs check out a commit hash from GitHub, never a local branch.
- The reference-transaction hook and the guard from the worktree record gain two rules: no ref update outside the run's own branch, and no pruning command from a run (new).

These are guards against mistakes, not isolation. A process that writes the mirror's files directly passes every hook, and a run's commits that it has not pushed then have no second copy. D3 puts that trade to the operator.

The record that says a clone's main worktree stays clean assumes a clone with a main worktree. A bare mirror has none, which satisfies the rule's intent and not its wording, so D8 asks to amend it.
