---
type: Design Appendix
title: tbhbdev Go port names and migration
description: The final name of every tbhbdev alpha command in the Go port, and how hooks, briefs, scripts, and agent-config move from the alpha names.
tags: [tbhbdev, go, names, migration, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words agy ITSM libexec -->

# tbhbdev Go port names and migration

Back to [the design](README.md).

Go has no `alpha` (D5). Each command takes its final name when it ports, and `tbhbdev alpha <command>` keeps working through the Python fallthrough until then. The commands below come from the `alpha` usage text at tools `938f65f` (existing, a local clone older than the outline's basis), and the review subcommands from `review.py` at `bf420ff`. `guard-ssh-agent`, `agent log`, `agent diff`, and `pr rebase` come from tools main at `1be1910`, and the `alpha issue` verbs from tools main at `32c4042`. A command added since then gets a row before its wave starts, and the bundle test below fails on a Python subcommand with no row.

## The name map

One file in the monorepo, `factory/cli/names.toml`, holds this map with the first bundle version that carries each final name (new). Each bundle ships it, so the installer, the Python forwarders through `TBHBDEV_BUNDLE`, the caller check, and the Go `agent` group's passthrough list read the same file, and this table mirrors it.

| Today | Final name | Wave |
| --- | --- | --- |
| `alpha review draft`, `check`, `policy`, `post`, `dismiss`, `reply` | `review draft`, `check`, `policy`, `post`, `dismiss`, `reply` | 1 |
| `alpha agent exec`, `wait`, `status`, `kill`, `clean`, `keep`, `log`, `diff`, and `protocol`, which phase 1 adds | `agent exec`, `wait`, `status`, `kill`, `clean`, `keep`, `log`, `diff`, `protocol` | 2 |
| `alpha agent ssh-serve` | `agent ssh-serve` | 2 |
| `alpha agent-env` | `agent env` | 3 |
| `alpha gh-shim` | `agent gh-shim` | 3 |
| `alpha identity record`, `trailer`, `consume`, `check`, `stamp-docs` | `identity record`, `trailer`, `consume`, `check`, `stamp-docs` | 3 |
| `alpha verify-range` | `identity verify-range` | 4 |
| `alpha guard-gh-pr` | `hook gh-pr` | 3 |
| `alpha guard-op` | `hook op` | 3 |
| `alpha state-guard` | `hook state` | 3 |
| `alpha session-context` | `hook session-context` | 3 |
| `alpha guard-gh-poll` | `hook gh-poll` | 3 |
| `alpha guard-gh-config-dir` | `hook gh-config-dir` | 3 |
| `alpha guard-scripts` | `hook scripts` | 3 |
| `alpha guard-ssh-agent` | `hook ssh-agent` | 3 |
| `alpha pre-push-guard` | `hook pre-push` | 4 |
| `alpha pr check`, `create`, `edit`, `hook`, `backfill-opened`, `rerun`, `rebase`, `delete-branch`, `close` | `pr check`, `create`, `edit`, `hook`, `backfill-opened`, `rerun`, `rebase`, `delete-branch`, `close` | 3 |
| `alpha merge` | `pr merge` | 3 |
| `alpha pr-stats` | `pr stats` | 4 |
| `alpha gh-app` | `gh-app` | 3 |
| `alpha repo bootstrap` | `repo bootstrap` | 3 |
| `alpha loop` | `loop` | 3 |
| `alpha issue close`, `create`, `link-sub`, `unlink-sub`, `block`, `unblock` | `issue close`, `create`, `link-sub`, `unlink-sub`, `block`, `unblock` | 4 |
| `issue import`, a top-level Python name | `issue import` | 4 |
| `alpha issue close`, as a guarded transition | `issue close`, which moves an issue from In progress to Done only when every acceptance criterion has recorded evidence, records that evidence, and otherwise refuses | none yet |
| `alpha brief new`, `check` | `brief new`, `check` | 4 |
| `alpha install` | `install` | 4 |
| `alpha version-check` | `install check` | 4 |

Issue status is set only by the issue commands, through a state machine in code that refuses a disallowed transition, and only one agent run works on an issue at a time (decided by tbhb).

The body check in `pr check` adds one rule:

- **No closing keywords.** It fails on `close`, `closes`, `closed`, `fix`, `fixes`, `fixed`, `resolve`, `resolves`, or `resolved` followed by an issue reference. A pull request writes `Refs: #N` when it contributes to an issue and `Completes: #N` when it finishes one, and the reviewer checks a `Completes:` against the issue's acceptance criteria (decided by tbhb).

The top-level names `admin`, `secret`, `adr`, `agy`, and `issue` keep their names and port in wave 4 ([appendix H](H-waves.md)). `cue` does not port, since Go runs no `cue` binary (D20). `analytics` keeps its name and runs through the Python fallthrough until it ports in a later wave. `host` is retired without a port (D2).

From the phase 1 release, the Python package also answers the top-level name `agent` with the `alpha agent` code, plus `agent env` and `agent gh-shim` with the `alpha agent-env` and `alpha gh-shim` code. Phase 1 needs it for `agent protocol`, which drivers in both languages call by the final name ([appendix I](I-protocol.md#choosing-a-version-per-launch)). The Go review code then starts runs by the final name, and the Go `agent` group takes that name in wave 2 ([appendix H](H-waves.md#review-runs-before-the-launcher-ports)).

The Go `agent` group takes the name in wave 2 before `env` and `gh-shim` port in wave 3. Its root command lists those two as passthrough subcommands, read from `names.toml`, and execs `$TBHBDEV_BUNDLE/bin/tbhbdev-py agent <subcommand> <argv>` for them, so no `agent` subcommand loses its answer (D26).

The review commands ship as the top-level `tbhbdev review` group in wave 1, two waves before `tbhbdev pr` (D21).

## Collisions with Python names

A Go command group takes a top-level name before Python does. Two rules keep that from hiding a Python command.

- **Built-in Python names.** A Go command group for `admin`, `secret`, `issue`, `agent`, or another top-level Python name ships only when it covers every Python subcommand of that name, in Go or by passthrough to `tbhbdev-py`. `tbhbdev issue` therefore ports `import` with every `alpha issue` verb, `close`, `create`, `link-sub`, `unlink-sub`, `block`, and `unblock`, together, and `tbhbdev agent` answers every subcommand that Python's `agent` answers, with `env` and `gh-shim` passed through until wave 3. A bundle test compares the Go group's subcommands and passthrough list with the Python ones from the pinned tools commit.
- **Reserved words.** `alpha`, `help`, and `version` are reserved from ported command groups, and the bundle build fails on them.

## The forwarder

When a Go command has shipped in one bundle, the next bundle turns its Python `alpha` entry into a forwarder. The first callers of the Go command are then the ones that chose its final name.

1. The forwarder runs only when `TBHBDEV_PATH` is set and `TBHBDEV_FORWARDED` is not. Otherwise it runs the Python code, so Python started outside a bundle still works.
2. It appends one line to `$XDG_STATE_HOME/tbhbdev/alpha-calls.log` with the time, the `alpha` name, and the parent process name.
3. It sets `TBHBDEV_FORWARDED=1` and replaces itself with `TBHBDEV_PATH` and the final name's argv.

It writes nothing to standard output or standard error, because hooks speak through those streams and a notice there would change what a harness sees.

## How callers move

| Caller | Where it lives | Commands it uses | Who moves it, and how |
| --- | --- | --- | --- |
| Harness hooks | The user-level harness settings that `tbhbdev alpha install` writes | the guards, `guard-ssh-agent` included, `state-guard`, `session-context`, `identity` | The installer reads `names.toml` and writes final names for every command its bundle carries |
| Git hooks | Each repository's hooks, installed by `tbhbdev alpha install` | `identity`, `pre-push-guard` | The same installer step |
| The forced command | Each host's authorized keys | `alpha agent ssh-serve` | The ops maintainer, host by host, in phase 3 ([appendix I](I-protocol.md#the-transition)) |
| Brief templates | `tbhbdev alpha brief new` in tools | `alpha pr check`, `alpha merge`, `alpha agent exec` | The tools maintainer, in the change after the forwarder ships |
| Briefs and prompts that sessions write | agent-config | any | The agent-config maintainer, one pull request per wave |
| Repository docs | `AGENTS.md`, runbooks, and designs in each repository, such as this wiki's `AGENTS.md` naming `tbhbdev alpha install` | any | Each repository's maintainer, one pull request per wave |
| Scripts | tbhb's scripts in `~/.local/libexec/tbhb` and the ops sync scripts | `version-check`, `gh-app`, `agent` | The ops maintainer |

For each command, in order:

1. **Ship.** The Go command ships in a bundle, and the fleet check shows every host on that bundle or later. The fleet check is `tbhbdev alpha version-check` (existing) until wave 4 ports it as `tbhbdev install check`.
2. **Forward.** The next bundle makes the Python `alpha` entry a forwarder.
3. **Move.** Each caller's owner changes the call to the final name. `rg 'tbhbdev alpha <command>'` across tools, tbhbdev, agent-config, ops, and wiki finds the work.
4. **Hold.** A planned lint in tbhb-dev/rules, fed the ported names from `names.toml`, fails on new `tbhbdev alpha <command>` text in any repository that runs the rules checks.
5. **Remove.** When every host's forwarder log shows no call for that command for 14 days, the Python code for it goes. `tbhbdev alpha <command>` then exits 2 with a usage message that names the final command.

Steps 3 and 4 run per wave, not per command, so each caller repository takes one pull request per wave.
