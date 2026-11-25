---
type: Design Appendix
title: tbhbdev Go port plugin contract
description: How the tbhbdev front routes a call, the contract every plugin binary follows, the shared Go library, and the version transition in which no launch is refused.
tags: [tbhbdev, go, plugins, contract, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words termio procexec buildinfo ghapi sshcall procgroup runfiles runstate statedir userconfig sigterm memfd syscall ITSM libexec -->

# tbhbdev Go port plugin contract

Back to [the design](README.md).

Everything here is new unless it names an existing file. [Appendix D](D-evidence.md) cites the existing behavior.

## Dispatch

The front is `cmd/tbhbdev`. Its `main` is one line that calls `front.Main` in `internal/dispatch/io/front` and exits with the result, as gh's `cmd/gh/main.go` does.

1. **Find the bundle.** `os.Executable`, then `filepath.EvalSymlinks`, gives the front's real path. Its directory's parent is the bundle root.
2. **Read the manifest.** One read of `<bundle>/manifest.json`. A missing or unreadable manifest exits 125 with one line on standard error.
3. **Route.** `dispatch.Resolve(argv, manifest, outside)` is pure and returns one route: a built-in, a bundle plugin, an outside plugin (only if D3 passes), or the Python fallthrough. The first word of argv picks it.
4. **Exec.** The front adds the contract variables to the environment and calls `syscall.Exec` on the route's absolute path. A failed exec exits 126.

| First word | Route | Notes |
| --- | --- | --- |
| none, `help`, `-h`, `--help` | built-in help | Lists bundle plugins from the manifest, then says other names go to Python |
| `help <name>` | `<name> --help` | Forwarded, so the plugin prints its own help |
| `version`, `--version` | built-in version | Prints the bundle version and commit from the stamp, with no file read |
| `alpha` | Python fallthrough, always | No Go plugin may take this name, and a bundle build fails if one does |
| a manifest plugin name | `<bundle>/bin/tbhbdev-<name>` | argv after the name, verbatim |
| an outside plugin name (D3) | the listed absolute path | Read only when no bundle plugin matches |
| anything else | Python fallthrough | Full argv, verbatim |

The front takes no flags of its own before the command word, so a flag such as `--repo` always reaches the plugin as typed. It starts no goroutine, reads no config, checks no network, and runs no migration. It doesn't hash or verify plugins at run time, because the installer did that once.

## The plugin contract

Contract version 1. A plugin is any executable the manifest lists, written in Go in this repository or, under D3, elsewhere.

| Part | Rule |
| --- | --- |
| Name | `tbhbdev-<name>`, where `<name>` matches `[a-z][a-z0-9-]*` and is one top-level command word |
| argv | `argv[0]` is the plugin's path, and the rest is the user's argv after `<name>`, unparsed, `--help` included |
| Reserved arguments | A lone `--version` prints the stamp. A first argument of `__complete` is kept for shell completion, and a plugin that doesn't support it exits 2 |
| Environment | Inherited unchanged, plus the variables below. A plugin reads them through `plugin.FromEnviron`, never by `os.Getenv` in pure code |
| Standard streams and descriptors | Inherited through exec: standard input, output, error, and every other open descriptor, such as the push-token pipe |
| Signals | Delivered straight to the plugin, since it holds the front's process ID |
| Exit status | The plugin's own (see below) |
| Writable state | Only under `$XDG_STATE_HOME/tbhbdev` and the run directories a command owns. Never the config directory |

| Variable | Value |
| --- | --- |
| `TBHBDEV_DISPATCHED` | `1`, so a plugin can tell the front started it |
| `TBHBDEV_CONTRACT` | `1`, the contract version |
| `TBHBDEV_PROG` | `tbhbdev <name>`, for usage messages |
| `TBHBDEV_PATH` | The front's resolved absolute path, for calling back into tbhbdev |
| `TBHBDEV_BUNDLE` | The bundle root's absolute path |
| `TBHBDEV_BUNDLE_VERSION` | The bundle version from the stamp |

A plugin never resolves its own path or guesses the front's. A test per variable checks that the front sets it and `plugin.FromEnviron` parses it.

### Exit codes

The plugin's status is the process's status, because the front no longer exists after exec. Plugins keep the codes the Python launcher uses today: 0, 2 for usage, 3, 4, 124 for a timeout, and 143 after SIGTERM, as the launcher's README tabulates (existing). They also use 1 for any other failure.

| Code | Meaning, from the front only |
| --- | --- |
| 125 | The front failed before routing: no manifest, or a malformed one |
| 126 | The route's file exists but exec failed |
| 127 | No route: the name isn't a plugin and the bundle has no Python fallthrough |

A plugin never exits 125, 126, or 127, and a test over each plugin's exit-code table checks it.

### The version stamp

`internal/buildinfo` holds `Version`, `Commit`, and `Date`, set by Bazel `x_defs` from the `STABLE_BUILD_SCM_*` workspace status keys, as awrel's `internal/buildmeta` does (existing). Every binary links it, and nothing else in the module holds a version string.

The Python wheel in the same bundle takes the same version string from the same commit, so `tbhbdev --version` and the Python `version.line()` agree. A release test runs every binary in the bundle with `--version` and fails unless each prints the bundle version.

### The manifest

`<bundle>/manifest.json`, written by the bundle build:

```json
{
  "contract": 1,
  "version": "<bundle version>",
  "commit": "<full commit>",
  "plugins": [
    {"name": "agent", "kind": "go", "file": "bin/tbhbdev-agent", "sha256": "<hex>"}
  ],
  "python": {"interpreter": "py/bin/python3", "module": "tbhbdev", "wheel_sha256": "<hex>"}
}
```

Each name appears once, and the build fails on a duplicate, on `alpha`, `help`, or `version`, or on a Go plugin that shadows a Python top-level name without covering it ([appendix G](G-names-and-migration.md#collisions-with-python-names)). Every entry states its kind, and the installer never infers one.

## The Python fallthrough

The bundle carries the Python package from its own commit, installed by the installer into `<bundle>/py/` with uv, never into `~/.local/bin`. The front runs the bundled interpreter as `python3 -m tbhbdev <argv>` with `TBHBDEV_PROG=tbhbdev`, so nothing depends on a script's first line.

The Python dispatcher works as it does today, including the `tbhbdev.plugins` entry points that graph and the ITSM repository use. Python reads `TBHBDEV_PROG` for usage messages (plan step 2). Entry-point plugins that also supply config files keep doing so on the Python side, and how they reach Go commands is part of D3.

## Install and the bundle directory

- **Layout.** `~/.local/libexec/tbhbdev/<version>/` holds `manifest.json`, `bin/`, and `py/`. The link `~/.local/bin/tbhbdev` points at `<version>/bin/tbhbdev`.
- **Install.** Download the archive for the host's platform, check it as D1 decides, check every manifest hash, unpack to a temporary directory beside the target, install the wheel into `py/`, rename the directory into place, then replace the link with an atomic rename. A failure at any step leaves the old link and bundle as they were.
- **Keep.** The installer keeps the previous two bundles. A run that started under one keeps its files, since an upgrade never deletes the directory a link points at or the one before it.
- **Undo.** Point the link back at the previous version.
- **Forced command.** Hosts run `<agent home>/.local/bin/tbhbdev alpha agent ssh-serve` as their forced command today (existing). That path stays valid, because the link now leads to the front, which routes `alpha` to the bundled Python.
- **Moving off uv.** On a host that ran `uv tool install`, the first bundle install runs `uv tool uninstall tbhbdev` and then writes the link, so uv's script never shadows the front. Which tool writes host authorized-keys entries today is unverified, and plan step 4 finds out before it moves a forced command.

Platforms: darwin-arm64, linux-amd64, and linux-arm64 (unverified against the host list).

## The library surface

Each package below is pure unless its path has `io/`. Names follow D11, and none is `core` or `shell`.

| Package | Holds | Used by |
| --- | --- | --- |
| `internal/buildinfo` | The stamp: `Version`, `Commit`, `Date`, and `Get()` | Every binary |
| `internal/dispatch` | `Resolve(argv, Manifest, Outside) Route`, `Manifest` parsing from bytes, collision checks | The front, the bundle build test |
| `internal/dispatch/io/front` | `Main() int`: find the bundle, read the manifest, call `Resolve`, exec | `cmd/tbhbdev` only |
| `internal/plugin` | `Env` and `FromEnviron([]string) (Env, error)`, the exit-code constants, typed errors, and `ExitCode(error) int`, the one place errors become codes | Every plugin |
| `internal/plugin/io/run` | `Main(name string, fn func(Context) error) int`: parse the environment, answer `--version`, call `fn`, map the error | Every plugin's `cmd/` |
| `internal/termio` | `Streams{In, Out, Err}`, terminal flags, and `Test()` returning buffers | Plugins that print |
| `internal/procexec/io/command` | Running a child with an explicit environment, a context deadline, process-group kill, and injected `newCommand` and `lookPath` functions | Adapters that run tools |
| `internal/statedir/io/files` | Paths under `$XDG_STATE_HOME/tbhbdev`, atomic writes, and the forwarder log reader | Telemetry, the installer, `install check` |
| `internal/ghapi/io/client` | The one constructor for GitHub HTTP clients, with User-Agent and auth options | Wave 2 commands |

Domain packages follow the same split, such as `internal/agent/argv`, `internal/agent/request`, `internal/agent/remote`, and `internal/agent/runstate` as pure packages, and `internal/agent/io/sshcall`, `internal/agent/io/procgroup`, `internal/agent/io/runfiles`, and `internal/agent/io/app` as I/O. Shared plain types from the Python `model.py` go in `internal/agent/model`.

There's no shared dependency object. A plugin's `io/app` builds the adapters it uses, resolves context such as the repository and host once, and passes plain values to pure code. Settings and credentials are separate adapters. Telemetry writes to the state directory in-process and starts no child.

## Versions and the no-refusal transition

### Today

The driver calls `same_version` in `decisions/driver.py` and refuses a host whose `version.line()` differs from its own (existing). Releases ship to every host at once, so a half-upgraded fleet refuses runs. The request envelope already carries its own versions, `{"v": 1, "request": {"v": 2}}` (existing).

### Phase 1: the last lockstep release

A Python release turns the build-version comparison into a logged warning. The envelope versions that `ssh_serve.py` already reads become the only compatibility test. It ships the way every release ships today. Hosts still on the old build refuse each other during this one rollout, as they do during any release now, and nothing after it refuses on a build version.

### Phase 2: the bundle, with Python behind the front

Hosts move to the bundle one at a time. A host on phase 1 and a host on a bundle run the same Python launcher code, and a build mismatch only logs. The forced command reaches the bundled Python through the front.

### Phase 3: Go `agent` in the bundle

The bundle gains `tbhbdev-agent`. Go drivers and Go servers speak envelope `v` 1 and request `v` 2 only (D12).

- **Python driver, Go host.** After callers move, a host's forced command becomes `tbhbdev agent ssh-serve`. The Go server serves every request in the corpus. A request it can't serve, such as an unknown field or version, goes to the bundled Python server: the Go server starts it as a child, feeds it the bytes already read followed by the rest of standard input, passes the other descriptors, forwards SIGINT and SIGTERM, and exits with the child's status.
- **Go driver, Python host.** A host on phase 1 or 2 serves `v` 1 and `v` 2, which is all a Go driver sends.
- **Mixed builds.** The Go driver logs a build mismatch and proceeds, like the phase 1 Python driver.

A corpus of request envelopes lives in the tools tests directory. CI runs it four ways: Python and Go servers each against every envelope, and each driver's output checked against a corpus member. Changes to the Python launcher after wave 1 land in Go first, so the two can't drift apart.

### Phase 4: the Python server retires

When `tbhbdev install check` shows every host on a bundle with the forced command on Go, and the forwarder logs show no `alpha agent` calls for 14 days, the Python server goes. A later protocol change then reaches every host before any driver sends it.

At no phase does a build version refuse a launch, and at no phase does a host lack a server for the request a driver sends.
