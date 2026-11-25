---
type: Design Appendix
title: tbhbdev Go port plugin contract
description: How the tbhbdev front routes a call, the contract every plugin binary follows, the install layout, and the shared Go library.
tags: [tbhbdev, go, plugins, contract, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words selfupdate termio procexec buildinfo ghapi sshcall procgroup runfiles runstate statedir userconfig sigterm memfd syscall ITSM libexec cobra koanf cuelang apptoken unported -->

# tbhbdev Go port plugin contract

Back to [the design](README.md).

Everything here is new unless it names an existing file. [Appendix D](D-evidence.md) cites the existing behavior. Every Go path here sits under `<cli-root>` in tbhb-dev/tbhbdev ([the design](README.md#the-open-cli-folder)).

## Dispatch

The front is `<cli-root>/cmd/tbhbdev`. Its `main` is one line that calls `front.Main` in `<cli-root>/internal/dispatch/io/front` and exits with the result, as gh's `cmd/gh/main.go` does.

1. **Find the bundle.** `os.Executable`, then `filepath.EvalSymlinks`, gives the front's real path. Its directory's parent is the bundle root.
2. **Lease the bundle.** Before it reads any file in the bundle, the front opens `<bundle>/.in-use` and takes a shared lock on it, without close-on-exec. If the bundle was pruned in between, it resolves the link once more and retries once ([appendix J](J-self-update.md#runs-in-progress)).
3. **Read the manifest.** One read of `<bundle>/manifest.json`. A missing or unreadable manifest exits 125 with one line on standard error.
4. **Check for an update.** Unless the first word is `version`, `--version`, or `help`, the front reads the self-update cache and calls the pure `selfupdate.Decide`. Anything but `Run` execs the bundle's self-update helper, which does the network work under its budget ([appendix J](J-self-update.md)). If that exec fails, the front logs one warning and goes on to route the current build.
5. **Route.** `dispatch.Resolve(argv, manifest, outside)` is pure and returns one route: a built-in, a bundle plugin, an outside plugin (only if D3 passes), or `tbhbdev-py`. The first word of argv picks it.
6. **Exec.** The front adds the contract variables to the environment and calls `syscall.Exec` on the route's absolute path. A failed exec exits 126.

| First word | Route | Notes |
| --- | --- | --- |
| none, `help`, `-h`, `--help` | built-in help | Lists bundle plugins from the manifest, then says other names go to `tbhbdev-py` |
| `help <name>` | `<name> --help` | Forwarded, so the plugin prints its own help |
| `version`, `--version` | built-in version | Prints the bundle version and commit from the stamp, with no file read |
| `alpha` | `<bundle>/bin/tbhbdev-py`, always | No Go plugin may take this name, and a bundle build fails if one does |
| a manifest plugin name | `<bundle>/bin/tbhbdev-<name>` | argv after the name, verbatim |
| an outside plugin name (D3) | the listed absolute path | Read only when no bundle plugin matches |
| anything else | `<bundle>/bin/tbhbdev-py` | Full argv, verbatim |

The front takes no flags of its own before the command word and links no flag library, so a flag such as `--repo` always reaches the plugin as typed ([appendix K](K-go-stack.md#why-the-front-doesnt-use-cobra)). It starts no goroutine, reads no config, makes no network call, and runs no migration, and it reads only the manifest and the self-update cache. A failed self-update never stops it from routing.

It doesn't hash or verify plugins at run time, because the installer did that once.

## The plugin contract

Contract version 1. A plugin is any executable the manifest lists, written in Go in tbhb-dev/tbhbdev or, under D3, elsewhere.

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

`<cli-root>/internal/buildinfo` holds `Version`, `Commit`, `CommitTime`, and `Date`, set by Bazel `x_defs` from the `STABLE_BUILD_SCM_*` workspace status keys, as awrel's `internal/buildmeta` does (existing). Every binary links it, and nothing else in the module holds a version string.

The Python wheel in the bundle comes from the tools commit that the monorepo pins, so it keeps its own tools version. `tbhbdev version` prints the bundle version and monorepo commit, then the tools commit of `tbhbdev-py`, from the manifest. A release test runs every Go binary in the bundle with `--version` and fails unless each prints the bundle version, and runs `bin/tbhbdev-py --version` and fails unless it names the pinned tools commit.

### The manifest

`<bundle>/manifest.json`, written by the bundle build:

```json
{
  "contract": 1,
  "version": "<bundle version>",
  "commit": "<full commit>",
  "channel": "main",
  "plugins": [
    {"name": "agent", "kind": "go", "file": "bin/tbhbdev-agent", "sha256": "<hex>"}
  ],
  "helpers": [
    {"name": "self-update", "file": "bin/tbhbdev-self-update", "sha256": "<hex>"}
  ],
  "python": {"entry": "bin/tbhbdev-py", "tools_commit": "<full tools commit>", "wheel_sha256": "<hex>"}
}
```

Each name appears once, and the build fails on a duplicate, on `alpha`, `help`, `version`, or `self-update`, or on a Go plugin that shadows a Python top-level name without covering it ([appendix G](G-names-and-migration.md#collisions-with-python-names)). Every entry states its kind, and the installer never infers one. `channel` is `main` only for bundles the tbhb-dev/tbhbdev release job builds from main, and self-update acts only on those. `commit` is the monorepo commit, and D23's attestation check pins it.

## The Python fallthrough

The bundle carries the tools wheel from the commit that the monorepo pins, and the installer installs it into `<bundle>/py/` with uv as a relocatable environment, so the directory still works after its rename into place (`uv venv --relocatable`, unverified against the pinned uv). The wheel's `tbhbdev-py` entry point, the same code as its `tbhbdev` script, sits in the front's own folder as `<bundle>/bin/tbhbdev-py`, a relative link to `../py/bin/tbhbdev-py` (D25).

The front execs `<bundle>/bin/tbhbdev-py <argv>` by absolute path with `TBHBDEV_PROG=tbhbdev`. It never runs `python3 -m tbhbdev`, and never searches `PATH`. The wheel's `tbhbdev` script stays inside `py/bin`, off `PATH`, so it can't shadow the front.

The Python dispatcher works as it does today, including the `tbhbdev.plugins` entry points that graph and the ITSM repository use. Python reads `TBHBDEV_PROG` for usage messages (plan step 2). Entry-point plugins that also supply config files keep doing so on the Python side, and how they reach Go commands is part of D3.

A Go plugin that owns a word before all its subcommands port passes the unported ones to the same file, `$TBHBDEV_BUNDLE/bin/tbhbdev-py <word> <argv>`, as the `agent` plugin does in wave 2 (D26).

## Install and the bundle directory

- **Layout.** `~/.local/libexec/tbhbdev/<version>/` holds `manifest.json`, `.in-use`, `bin/` with the front, the plugins, the helper, and `tbhbdev-py`, and `py/`. The link `~/.local/bin/tbhbdev` points at `<version>/bin/tbhbdev`.
- **Install.** Download the archive for the host's platform, check it as D23 decides, check every manifest hash, unpack to a temporary directory beside the target, install the wheel into `py/`, rename the directory into place, recheck the link under the lock, then replace the link with an atomic rename ([appendix J](J-self-update.md#the-swap)). A failure at any step leaves the old link and bundle as they were.
- **Keep.** The installer keeps the link's target and the two bundles before it, and never deletes a bundle a run still holds open ([appendix J](J-self-update.md#runs-in-progress)).
- **Self-update and sync.** Both run this same install, under one lock, as [appendix J](J-self-update.md) describes.
- **Undo.** Point the link back at the previous version.
- **Forced command.** Hosts run `<agent home>/.local/bin/tbhbdev alpha agent ssh-serve` as their forced command today (existing). That path stays valid, because the link now leads to the front, which routes `alpha` to `tbhbdev-py`.
- **Moving off uv.** On a host that ran `uv tool install`, `~/.local/bin/tbhbdev` is uv's script. The first bundle install never removes it before the front exists. It checks and places the bundle first, then renames the new link over uv's script, which replaces it in one step, so the path always runs one of the two. Only then does it remove uv's tool environment. Whether `uv tool uninstall` would also delete the new link at that path is unverified, so the installer deletes the environment directory that `uv tool dir` names and leaves `~/.local/bin` alone. It then checks that `~/.local/bin/tbhbdev` resolves to the new front, and writes the link again if not. A failure before the rename leaves the uv install as it was. Which tool writes host authorized-keys entries today is unverified, and plan step 4 finds out before it moves a forced command.

Platforms: darwin-arm64, linux-amd64, and linux-arm64 (unverified against the host list).

## The library surface

Each package below sits under `<cli-root>`, and is pure unless its path has `io/`. Names follow D11, and none is `core` or `shell`.

| Package | Holds | Used by |
| --- | --- | --- |
| `internal/buildinfo` | The stamp: `Version`, `Commit`, `CommitTime`, `Date`, and `Get()` | Every binary |
| `internal/dispatch` | `Resolve(argv, Manifest, Outside) Route`, `Manifest` parsing from bytes, collision checks | The front, the bundle build test |
| `internal/dispatch/io/front` | `Main() int`: find the bundle, read the manifest and the self-update cache, call `Decide` and `Resolve`, exec | `cmd/tbhbdev` only |
| `internal/selfupdate` | `Decide(Build, Cache, now, Policy) Action` and cache parsing from bytes | The front, the self-update helper |
| `internal/selfupdate/io/swap` | The pointer fetch, download, checks, rename, link swap, pruning, and re-exec | `cmd/tbhbdev-self-update`, the installer, sync |
| `internal/agent/protocol` | The embedded schema, `Validate` through `cuelang.org/go`, the lenient reader, and `Choose` ([appendix I](I-protocol.md)) | The agent driver and server, the version check |
| `internal/schemas` | The embedded Configure Unify Execute (CUE) and JSON Schema files, and `Validate(name, document)` through `cuelang.org/go`, returning findings ([appendix K](K-go-stack.md#schemas-with-cue-in-go)) | The review commands, the content check, and later the issue and pull request commands |
| `internal/plugin` | `Env` and `FromEnviron([]string) (Env, error)`, the exit-code constants, typed errors, and `ExitCode(error) int`, the one place errors become codes | Every plugin |
| `internal/plugin/io/run` | `Main(name string, newRoot func(Context) *cobra.Command) int`: parse the environment, answer `--version`, build the cobra root, execute it with the argv tail, and map the error | Every plugin's `cmd/` |
| `internal/config` | The plain `Config` struct and `Check`, which names each missing `[table] key` | Pure code that reads settings |
| `internal/config/io/load` | One koanf load of `$XDG_CONFIG_HOME/tbhbdev/config.toml`, at most once per process, into `config.Config` | Each plugin's `io/app` |
| `internal/termio` | `Streams{In, Out, Err}`, terminal flags, and `Test()` returning buffers | Plugins that print |
| `internal/procexec/io/command` | Running a child with an explicit environment, a context deadline, process-group kill, and injected `newCommand` and `lookPath` functions. It refuses any program whose base name is `cue` (D20) | Adapters that run tools |
| `internal/statedir/io/files` | Paths under `$XDG_STATE_HOME/tbhbdev`, atomic writes, and the forwarder log reader | Telemetry, the installer, `install check` |
| `internal/ghapi/io/client` | The one go-github client, built with an injected transport, token, User-Agent, and timeout (D24) | The wave 1 review commands, then every GitHub command |
| `internal/ghapi/io/graphql` | The small GraphQL helper on the same client: ready for review, the merge queue state and enqueue, and Projects v2 by node ID | The pull request and project commands |
| `internal/ghapi/io/apptoken` | App installation tokens from a standard-library JWT and go-github's installation calls, with the repository-then-organization lookup, cache, cooldown, and secret-free errors | The review poster, then the `gh-app` commands |

Domain packages follow the same split, such as `internal/agent/argv`, `internal/agent/request`, `internal/agent/remote`, and `internal/agent/runstate` as pure packages, and `internal/agent/io/sshcall`, `internal/agent/io/procgroup`, `internal/agent/io/runfiles`, and `internal/agent/io/app` as input/output (I/O). Shared plain types from the Python `model.py` go in `internal/agent/model`.

There's no shared dependency object. A plugin's `io/app` builds the adapters it uses, resolves context such as the repository and host once, and passes each cobra command only what it uses ([appendix K](K-go-stack.md#explicit-dependencies-and-no-shared-object)). Settings and credentials are separate adapters. Telemetry writes to the state directory in-process and starts no child.

## Versions

The driver and the host agree on a protocol version, the version of one schema file, and never on a build. [Appendix I](I-protocol.md) gives the schema, its version rules, the CI check that holds them, and the transition in which no launch is refused. A plugin from outside the bundle (D3) whose contract version differs gets a logged warning and still runs.
