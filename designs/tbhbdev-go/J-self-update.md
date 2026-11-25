---
type: Design Appendix
title: tbhbdev Go port self-update
description: How tbhbdev updates itself at start when it's behind main, with a pure decision, an atomic swap, no refusal on failure, and how it relates to event-driven sync.
tags: [tbhbdev, go, install, self-update, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words selfupdate libexec flock -->

# tbhbdev Go port self-update

Back to [the design](README.md).

Everything here is new (D15). It starts with the bundle in plan step 5. Machines still on the phase 1 Python install have no self-update and rely on event-driven sync.

## The shape

The front decides, and a separate bundle binary acts. The front does one file read and one pure call, so it still does no network work and starts no process before its exec ([appendix E](E-plugin-contract.md#dispatch)).

1. **Read.** The front reads the cache file. A missing or unreadable cache is a stale one.
2. **Decide.** `selfupdate.Decide` in `internal/selfupdate` is pure and returns `Run`, `Refresh`, or `Update(commit)`.
3. **Run.** On `Run`, dispatch goes on as [appendix E](E-plugin-contract.md#dispatch) describes.
4. **Hand off.** On `Refresh` or `Update`, the front execs `<bundle>/bin/tbhbdev-self-update` with the original argv. That binary is the I/O adapter: it does the network call, the download, the checks, the rename, and the re-exec.
5. **Re-exec.** The adapter always ends by exec of a front, the new one after a swap or the current one otherwise, with `TBHBDEV_SELF_UPDATED=1` set. The front drops that variable before dispatch, so the next call checks again.

`tbhbdev version`, `--version`, and `help` skip the check, so they still read only the manifest. `tbhbdev-self-update` is a helper the manifest lists apart from plugins, and no plugin may take the name `self-update`.

## The decision

`Decide(build, cache, now, policy)` takes plain values: the build commit and its commit time from the stamp, the manifest's channel, the cached main commit with its commit time, the cache's fetch and failure times, the clock as a value, and the policy. The policy holds the time to live, 10 minutes, and the two environment flags.

| In this order | Returns |
| --- | --- |
| `TBHBDEV_NO_SELF_UPDATE=1`, or `TBHBDEV_SELF_UPDATED=1` | `Run` |
| The channel isn't `main`, as for a developer build or a branch bundle | `Run` |
| A failure is recorded less than one time to live ago | `Run` |
| The cache is missing, unreadable, or older than one time to live | `Refresh` |
| The build commit equals the main commit, or its commit time isn't earlier | `Run` |
| Otherwise | `Update(main commit)` |

After a refresh, the adapter writes the cache and calls `Decide` again, which can then return only `Run` or `Update`. Table tests cover each row and each boundary of the time to live.

## The cache

The cache is `$XDG_CACHE_HOME/tbhbdev/main.json`, or `~/.cache/tbhbdev/main.json` when that variable is unset. It holds the main commit, its commit time and bundle version, the fetch time, and the last failure with its time and reason. Deleting it is always safe, since the next start refreshes it.

A refresh is one HTTPS GET with a 2-second deadline of the release pointer, a small `main.json` the release job publishes beside each main bundle. It names the commit, its commit time, the version, and each platform's archive with its Secure Hash Algorithm 256 (SHA-256) hash. With a 10-minute time to live, a machine makes at most six such calls an hour, and most starts make none.

## The swap

The install keeps the layout in [appendix E](E-plugin-contract.md#install-and-the-bundle-directory): versioned directories under `~/.local/libexec/tbhbdev/`, and one link, `~/.local/bin/tbhbdev`. No step writes into a file or directory that a run may be using.

1. Take `~/.local/libexec/tbhbdev/.update.lock` with a non-blocking exclusive `flock`. If another run or sync holds it, record nothing and run the current build.
2. Download the archive into a new `.tmp-<random>/` directory beside the versioned ones, on the same file system, with a 60-second deadline.
3. Check the archive's SHA-256 hash against the pointer and its provenance as D1 decides. Unpack it, check every manifest hash, and install the wheel into `py/`.
4. Rename `.tmp-<random>/` to `<version>/`. If `<version>/` already exists, because sync or another run got there first, delete the temporary directory and use the existing one.
5. Create a new link `~/.local/bin/.tbhbdev-<random>` to `<version>/bin/tbhbdev`, then rename it over `~/.local/bin/tbhbdev`. The link always exists, and it names either the old bundle or the new one.
6. Write the cache, release the lock, and exec the new front by its absolute path with the original argv.

### Runs in progress

Each run keeps its own bundle open until it exits. The front opens `<bundle>/.in-use` and takes a shared `flock` on it, without close-on-exec, so the plugin, the Python fallthrough, and their children hold it through every exec.

Pruning keeps the link's target and the two before it. It removes an older directory only after it takes a non-blocking exclusive lock on that directory's `.in-use`, so it never deletes a bundle a run still uses. A temporary directory more than an hour old is removed under the update lock, since a killed download leaves one behind.

## Failures

No failure refuses a run. Each one writes one warning line to `$XDG_STATE_HOME/tbhbdev/self-update.log`, and also to standard error when standard error is a terminal. Hooks speak through their streams, so a hook never sees the warning.

The failure goes into the cache, so the next try waits one time to live.

| Failure | What happens |
| --- | --- |
| No network, or the 2-second refresh deadline passes | Warn, record, run the current build |
| The download fails or passes its 60-second deadline | Warn, delete the temporary directory, record, run the current build |
| A bad SHA-256 hash, a failed provenance check, or a manifest hash mismatch | Warn with both hashes, delete the temporary directory, record, run the current build |
| A read-only install, a full disk, or a failed wheel install | Warn, record, run the current build. The adapter tests that the install directory is writable before any network call |
| The cache can't be written | Warn and run the current build. Each start then refreshes, so such a machine sets `TBHBDEV_NO_SELF_UPDATE=1` |
| The lock is held | Run the current build, with no warning |
| The exec of the new front fails | Warn and exec the current front. If that fails too, exit 126 as the front does |

A bad bundle on main reaches machines as fast as a good one. Prior bundles stay on disk, and a revert on main ships forward within one release. `TBHBDEV_NO_SELF_UPDATE=1` holds a machine where it is.

## Event-driven sync

The tools maintainer is building event-driven sync, which installs the new release on each machine when a pull request merges to main (unverified beyond the brief). Sync is the main path, and self-update is the fallback.

- **One install path.** Sync runs the same install code, takes the same lock, and makes the same swap. Whichever comes second finds the directory present and only moves the link if needed.
- **Sync writes the cache.** After an install, sync writes the cache with the installed commit and the current time. The next start's `Decide` returns `Run` with no network call.
- **Self-update covers the gaps.** A machine that was asleep, offline, newly built, or failed its sync catches up at its next start. That keeps every machine within reach of the protocol's one-version window ([appendix I](I-protocol.md#additive-and-breaking-changes)).

So most starts read one small file and run. A start that updates is one that sync missed.
