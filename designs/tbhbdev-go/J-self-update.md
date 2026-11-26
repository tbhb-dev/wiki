---
type: Design Appendix
title: tbhbdev Go port self-update
description: How tbhbdev updates itself at start when it is behind main, with a pure decision, a time budget, an atomic swap under leases, no blocked dispatch on failure, and how it relates to event-driven sync.
tags: [tbhbdev, go, install, self-update, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words selfupdate libexec flock setsid -->

# tbhbdev Go port self-update

Back to [the design](README.md).

Everything here is new (D15). It starts with the bundle in plan step 5. Machines still on the phase 1 Python install have no self-update and rely on event-driven sync.

## The shape

The front decides, and the same Go binary handles the update in an internal mode. The front reads the self-update cache and makes one pure call for that decision. It does no network work and starts no process before its exec. [Dispatch](README.md#dispatch)

The separate helper executable and its manifest entry were replaced by the 2026-11-26 no-plugins decision (D3).

1. **Lease.** The front takes a shared lease on its own bundle before it reads any file in it ([runs in progress](#runs-in-progress)).
2. **Read.** The front reads the manifest and the cache file. A missing or unreadable cache is a stale one.
3. **Decide.** `selfupdate.Decide` in `factory/cli/internal/selfupdate` is pure and returns `Run`, `Refresh`, or `Update(commit)`.
4. **Run.** On `Run`, the front continues to route the command. [Dispatch](README.md#dispatch)
5. **Hand off.** On `Refresh` or `Update`, the front execs `<bundle>/bin/tbhbdev` in its internal update mode with the original argv. That mode runs the input/output (I/O) adapter: it does the network call, the download, the checks, the rename, and the re-exec. If the exec of the helper fails, the front writes one warning to the log and dispatches the current build, as on `Run`.
6. **Re-exec.** The adapter always ends by exec of a front, the new one after an in-band swap or the current one otherwise, with `TBHBDEV_SELF_UPDATED=1` set. The front drops that variable before dispatch, so the next call checks again. After a swap, the new front also closes the update lock it inherited, named by `TBHBDEV_UPDATE_LOCK_FD`, once it holds its lease and has read its manifest, and drops that variable too.

`tbhbdev version`, `--version`, and `help` skip the check, so they still read only the manifest.

## The decision

`Decide(build, cache, now, policy)` takes plain values: the build commit and its release number from the stamp, the manifest's channel, the cached main commit with its release number, the cache's fetch and failure times, the clock as a value, and the policy. The policy holds the time to live, 10 minutes, the in-band budget, 3 seconds, and the three environment flags.

| In this order | Returns |
| --- | --- |
| `TBHBDEV_NO_SELF_UPDATE=1`, or `TBHBDEV_SELF_UPDATED=1` | `Run` |
| The channel is not `main`, as for a developer build or a branch bundle | `Run` |
| `TBHBDEV_UPDATE_NOW=1`, which only the protocol's no-common-version path sets ([appendix I](I-protocol.md#choosing-a-version-per-launch)) | `Refresh` |
| A failure is recorded less than one time to live ago | `Run` |
| The cache is missing, unreadable, or older than one time to live | `Refresh` |
| The build commit equals the main commit, or its release number is not lower | `Run` |
| Otherwise | `Update(main commit)` |

After a refresh, the adapter writes the cache and calls `Decide` again with `TBHBDEV_UPDATE_NOW` dropped, which can then return only `Run` or `Update`. Table tests cover each row and each boundary of the time to live.

### Release order

A bundle's release number orders releases, never its commit time. The release job stamps each main bundle with the count of first-parent commits on main up to its commit, `git rev-list --count --first-parent`, in the manifest and the pointer. Main lands only through the merge queue and is never rewritten, so each later main commit has a higher number, even when its commit time is equal or earlier, as a rebased or queued commit's can be. A table test holds two main commits with the same commit time and one with an earlier time than its parent, and `Decide` returns `Update` for each newer one.

## The cache

The cache is `$XDG_CACHE_HOME/tbhbdev/main.json`, or `~/.cache/tbhbdev/main.json` when that variable is unset. It holds the main commit, its release number and bundle version, the fetch time, and the last failure with its time and reason. Deleting it is always safe, since the next start refreshes it.

A refresh is one HTTPS GET of the release pointer, a small `main.json` the tbhb-dev/tbhbdev release job publishes beside each main bundle. It names the commit, its release number, the version, and each platform's archive with its Secure Hash Algorithm 256 (SHA-256) hash. With a 10-minute time to live, a machine makes at most six such calls an hour, and most starts make none.

## The budget

Agent hooks run `tbhbdev` on every tool call and wait on it under a timeout, so update work in the foreground has a hard limit. The adapter starts a 3-second budget clock when it begins, and every in-band step takes its deadline from what remains.

- **Refresh.** The pointer GET gets at most 2 seconds of the budget.
- **In band.** The download, the checks, the unpack, the wheel install, and the swap run in the foreground only while budget remains. When they finish in time, the adapter execs the new front.
- **Past the budget.** The adapter cancels its in-band work, deletes its temporary directory, and starts the detached installer for that commit by invoking the same Go binary in a new session through `setsid`, with standard streams on `/dev/null` and no inherited descriptor, so it holds no hook pipe, no push-token pipe, and no lease. It then releases the update lock if it holds it and execs the current front, so the caller's run starts within the budget plus one exec. The budget is checked only before the publish step of [the swap](#the-swap), so a swap that has moved the link always finishes its handoff. The proof step runs before that check, so a slow proof sends the work to the detached installer too.
- **The detached installer.** It takes the update lock, downloads under a 60-second deadline, checks, installs, proves the target runs, and publishes as [the swap](#the-swap) describes, writes the cache, releases the lock, and exits. It never execs a front, so the proof in step 7 of the swap is what keeps a bundle that cannot run off the public link. The next start runs the new bundle.

The harness hooks that `alpha install` writes run under a 10-second timeout (existing, `HARNESS_TIMEOUT` at tools `1be1910`), so a 3-second budget plus one exec leaves the hook's own work most of its time. The budget is a policy value, and plan step 5 checks it against the hook settings in force then.

## The swap

The install keeps versioned directories under `~/.local/libexec/tbhbdev/` and one link, `~/.local/bin/tbhbdev`. No step writes into a file or directory that a run may be using.

1. **Lock.** Take `~/.local/libexec/tbhbdev/.update.lock` with a non-blocking exclusive `flock`. If another run or sync holds it, record nothing and run the current build.
2. **Download.** Fetch the archive into a new `.tmp-<random>/` directory beside the versioned ones, on the same file system.
3. **Check.** Check the archive's SHA-256 hash against the pointer and its attestation as D23 pins it. Unpack it, check every manifest hash, check that the manifest's commit equals the pointer's, and install the wheel into `py/` as a relocatable environment.
4. **Place.** Rename `.tmp-<random>/` to `<version>/`. If `<version>/` already exists, because sync or another run got there first, delete the temporary directory and use the existing one.
5. **Recheck.** Still under the lock, read the link's current target and its manifest. If that target's commit is the new one or later by release number, because sync published it meanwhile, leave the link alone and use the current target. The swap never publishes a bundle older than the one the link names.
6. **Lease the target.** Open `<target>/.in-use` and take a shared lease on it, without close-on-exec, before anything else can prune.
7. **Prove it runs.** Still under the lock, run `<target>/bin/tbhbdev version` and `<target>/bin/tbhbdev-py --version` as children, with `TBHBDEV_NO_SELF_UPDATE=1` and a deadline from the budget. Both must exit 0 and report the target's monorepo commit. The Go command must also print the target's bundle version. On any other result, record the failure, delete the target if this run placed it, and run the current build. No public link ever names a target that has not run.
8. **Publish.** Create a new link `~/.local/bin/.tbhbdev-<random>` to `<target>/bin/tbhbdev`, then rename it over `~/.local/bin/tbhbdev`. The link always exists, and it names either the old bundle or a new one that has run. Remember the old target.
9. **Hand off.** Write the cache, and exec the new front by its absolute path with the original argv, still holding the update lock. The lock's descriptor and the lease from step 6 pass through the exec, without close-on-exec, and `TBHBDEV_UPDATE_LOCK_FD` names the lock's. The new front closes that descriptor once it holds its own lease and has read its manifest, before dispatch, so the command starts with the lock released. The lock is never released between the publish and a successful start of the new front.

### A failed handoff

If the exec in step 9 fails, the adapter still holds the update lock, since a failed exec leaves the process and its descriptors in place. No other update or sync can move the link meanwhile, so it renames a link to the old target back over `~/.local/bin/tbhbdev`, records the failure in the cache, releases the lock, and execs the current front. The target already ran in step 7, so a caller that resolves the link in this short window runs a bundle that runs.

### Runs in progress

Each run leases its own bundle until it exits. The front opens `<bundle>/.in-use` and takes a shared `flock` on it before it reads the manifest, without close-on-exec, so Go commands, `tbhbdev-py`, and their children hold it through every exec.

- **A bundle pruned before the lease.** The front resolves the link, then takes the lease. If `<bundle>/.in-use` is gone, or the lease succeeds but the manifest is gone, a pruner got there first. The front resolves the link once more and retries once. A second miss exits 125.
- **Pruning.** It runs under the update lock and keeps the link's target, the two before it, and every protocol bridge ([appendix I](I-protocol.md#bridges)). It removes an older directory only after it takes a non-blocking exclusive lock on that directory's `.in-use`. While it still holds that lock, it renames the directory to `.trash-<random>` and only then deletes it, so a front that gets its lease afterward finds no manifest and retries.
- **Temporary directories.** A `.tmp-` or `.trash-` directory more than an hour old is removed under the update lock, since a killed download leaves one behind.

## Failures

No failure blocks dispatch. Each one writes one warning line to `$XDG_STATE_HOME/tbhbdev/self-update.log`, and also to standard error when standard error is a terminal. Hooks speak through their streams, so a hook never sees the warning.

The failure goes into the cache, so the next try waits one time to live.

| Failure | What happens |
| --- | --- |
| The exec of the self-update helper fails | The front warns and dispatches the current build |
| No network, or the refresh deadline passes | Warn, record, run the current build |
| The in-band budget runs out | Start the detached installer, run the current build. Not a failure, so nothing is recorded |
| The detached installer's download fails or passes its 60-second deadline | Warn, delete the temporary directory, record |
| A bad SHA-256 hash, a failed attestation check, a commit mismatch, or a manifest hash mismatch | Warn with both values, delete the temporary directory, record, run the current build |
| The new bundle fails its proof run | Warn with the command and its status, delete the target if this run placed it, record, leave the link alone, run the current build |
| The recheck finds the link at the new commit or later | Use the current target, with no warning |
| A read-only install, a full disk, or a failed wheel install | Warn, record, run the current build. The adapter tests that the install directory is writable before any network call |
| The cache cannot be written | Warn and run the current build. Each start then refreshes, so such a machine sets `TBHBDEV_NO_SELF_UPDATE=1` |
| The lock is held | Run the current build, with no warning |
| The exec of the new front fails | Restore the old link as [a failed handoff](#a-failed-handoff) describes, warn, and exec the current front. If that fails too, exit 126 as the front does |

A bad bundle on main reaches machines as fast as a good one. Prior bundles stay on disk, and a revert on main ships forward within one release. `TBHBDEV_NO_SELF_UPDATE=1` holds a machine where it is.

## Tests

Each row below is a test in the self-update slice. Pure rows are `unit_go_test` tables, and the rest are `io_go_test` cases in a temporary root with a fake pointer server.

| Test | Passes when |
| --- | --- |
| `Decide` table | Each row above, each time-to-live boundary, and `TBHBDEV_UPDATE_NOW` return the listed action |
| Helper exec fails | When exec of the update mode fails, the front dispatches the current build and exits with the command's status |
| Slow download | With a server that stalls, the caller's command starts within the budget plus 500 ms, a detached installer runs, and the hook's standard output pipe closes when the command exits |
| Detached installer descriptors | The detached process holds no descriptor of the caller's, checked through `/proc` or `lsof` |
| Failed handoff | With a new front that cannot exec, the link names the old target afterward and the current build runs, and the lock was never released between the publish and the restore |
| Proof before publish | With a target whose front or `tbhbdev-py` exits nonzero, in band and in the detached installer, the link never names it |
| Publication under concurrency | While callers run `tbhbdev version` through the link in a loop, a swap whose proof fails, one whose handoff exec fails, and one that succeeds each leave every caller with exit 0 from a bundle that ran, and a second updater started meanwhile never takes the lock |
| Release order | A newer main commit with an equal or earlier commit time is installed, and an older one with a later commit time is not |
| Lease before read | A pruner that runs between the link read and the lease makes the front retry once and run, never read a half-deleted bundle |
| Lease across the handoff | A prune started right after the new front closes the inherited lock never removes the new target before the new front runs |
| No older publish | When sync publishes a newer bundle during a slow update, the slow update leaves the link on the newer one |
| Two swaps at once | One wins the lock, the other runs its current build, and the link ends on one complete bundle |
| Each failure row | Each failure in the table above runs the current build, writes one log line, and records the failure where the table says so |

## Event-driven sync

The design uses event-driven sync to install the new release on each machine when a pull request merges to main in tbhb-dev/tbhbdev (D22). Sync is the main path, and self-update is the fallback.

- **One install path.** Sync runs the same install code, takes the same lock, and makes the same swap with the same recheck. Whichever comes second finds the directory present and moves the link only if its bundle is newer.
- **Sync writes the cache.** After an install, sync writes the cache with the installed commit and the current time. The next start's `Decide` returns `Run` with no network call.
- **Self-update covers the gaps.** A machine that was asleep, offline, newly built, or failed its sync catches up at its next start. That keeps every machine within reach of the protocol's one-version window ([appendix I](I-protocol.md#additive-and-breaking-changes)).

So most starts read one small file and run. A start that updates is one that sync missed.
