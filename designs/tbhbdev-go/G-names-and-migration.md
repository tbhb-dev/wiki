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

Go has no `alpha` (D5). Each command takes its final name when it ports, and `tbhbdev alpha <command>` keeps working through the Python fallthrough until then. The commands below come from the `alpha` usage text at tools `938f65f` (existing, a local clone older than the outline's basis), and the review subcommands from `review.py` at `bf420ff`. `guard-ssh-agent`, `agent log`, `agent diff`, and `pr rebase` come from tools main at `1be1910`. A command added since then gets a row before its wave starts, and the bundle test below fails on a Python subcommand with no row.

## The name map

One file in the monorepo, `<cli-root>/names.toml`, holds this map with the first bundle version that carries each final name (new). Each bundle ships it, so the installer, the Python forwarders through `TBHBDEV_BUNDLE`, the caller check, and the `agent` plugin's passthrough list read the same file, and this table mirrors it.

| Today | Final name | Plugin binary | Wave |
| --- | --- | --- | --- |
| `alpha review draft`, `check`, `policy`, `post`, `dismiss`, `reply` | `review draft`, `check`, `policy`, `post`, `dismiss`, `reply` | `tbhbdev-review` | 1 |
| `alpha agent exec`, `wait`, `status`, `kill`, `clean`, `keep`, `log`, `diff` | `agent exec`, `wait`, `status`, `kill`, `clean`, `keep`, `log`, `diff` | `tbhbdev-agent` | 2 |
| `alpha agent ssh-serve` | `agent ssh-serve` | `tbhbdev-agent` | 2 |
| `alpha agent-env` | `agent env` | `tbhbdev-agent`, by passthrough to `tbhbdev-py` from wave 2 (D26) | 3 |
| `alpha gh-shim` | `agent gh-shim` | `tbhbdev-agent`, by passthrough to `tbhbdev-py` from wave 2 (D26) | 3 |
| `alpha identity record`, `trailer`, `consume`, `check`, `stamp-docs` | `identity record`, `trailer`, `consume`, `check`, `stamp-docs` | `tbhbdev-identity` | 3 |
| `alpha verify-range` | `identity verify-range` | `tbhbdev-identity` | 4 |
| `alpha guard-gh-pr` | `hook gh-pr` | `tbhbdev-hook` | 3 |
| `alpha guard-op` | `hook op` | `tbhbdev-hook` | 3 |
| `alpha state-guard` | `hook state` | `tbhbdev-hook` | 3 |
| `alpha session-context` | `hook session-context` | `tbhbdev-hook` | 3 |
| `alpha guard-gh-poll` | `hook gh-poll` | `tbhbdev-hook` | 3 |
| `alpha guard-gh-config-dir` | `hook gh-config-dir` | `tbhbdev-hook` | 3 |
| `alpha guard-scripts` | `hook scripts` | `tbhbdev-hook` | 3 |
| `alpha guard-ssh-agent` | `hook ssh-agent` | `tbhbdev-hook` | 3 |
| `alpha pre-push-guard` | `hook pre-push` | `tbhbdev-hook` | 4 |
| `alpha pr check`, `create`, `edit`, `hook`, `backfill-opened`, `rerun`, `rebase`, `delete-branch`, `close` | `pr check`, `create`, `edit`, `hook`, `backfill-opened`, `rerun`, `rebase`, `delete-branch`, `close` | `tbhbdev-pr` | 3 |
| `alpha merge` | `pr merge` | `tbhbdev-pr` | 3 |
| `alpha pr-stats` | `pr stats` | `tbhbdev-pr` | 4 |
| `alpha gh-app` | `gh-app` | `tbhbdev-gh-app` | 3 |
| `alpha repo bootstrap` | `repo bootstrap` | `tbhbdev-repo` | 3 |
| `alpha loop` | `loop` | `tbhbdev-loop` | 3 |
| `alpha issue close` | `issue close` | `tbhbdev-issue` | 4 |
| `alpha brief new`, `check` | `brief new`, `check` | `tbhbdev-brief` | 4 |
| `alpha install` | `install` | `tbhbdev-install` | 4 |
| `alpha version-check` | `install check` | `tbhbdev-install` | 4 |

The top-level names `admin`, `secret`, `adr`, `agy`, and `issue` keep their names and port in wave 4 ([appendix H](H-waves.md)). `cue` doesn't port, since Go runs no `cue` binary (D20), and `analytics` and `host` follow D2.

From wave 1, the Python package also answers the top-level name `agent` with the `alpha agent` code, plus `agent env` and `agent gh-shim` with the `alpha agent-env` and `alpha gh-shim` code. The Go review code then starts runs by the final name, and the Go `agent` plugin takes that name in wave 2 ([appendix H](H-waves.md#review-runs-before-the-launcher-ports)).

The wave 2 plugin owns the whole word `agent` before `env` and `gh-shim` port in wave 3. Its root command lists those two as passthrough subcommands, read from `names.toml`, and execs `$TBHBDEV_BUNDLE/bin/tbhbdev-py agent <subcommand> <argv>` for them, so no `agent` subcommand loses its answer (D26).

The harness hooks share one binary, `tbhbdev-hook`, because they run on every tool call and a small binary starts fastest. The pull request commands share `tbhbdev-pr` because they share the GitHub adapters. The review commands get their own `tbhbdev-review`, because they ship in wave 1, two waves before `pr` (D21).

## Collisions with Python names

A Go plugin takes a name before Python does. Three rules keep that from hiding a Python command:

- **Built-in Python names.** A Go plugin for `admin`, `secret`, `issue`, `agent`, or another top-level Python name ships only when it covers every Python subcommand of that name, in Go or by passthrough to `tbhbdev-py`. `tbhbdev-issue` therefore ports `import` and `close` together, and `tbhbdev-agent` answers every subcommand that Python's `agent` answers, with `env` and `gh-shim` passed through until wave 3. A bundle test compares the Go plugin's subcommands and passthrough list with the Python ones from the pinned tools commit.
- **Entry-point plugins from other repositories.** graph and the private ITSM repository add names through `tbhbdev.plugins`. The bundle can't see those, so the fleet check lists each host's entry-point names and reports any that a bundle plugin shadows. Until wave 4 that is a new report in `tbhbdev alpha version-check`, added before the first Go plugin ships.
- **Reserved words.** `alpha`, `help`, and `version` never name a Go plugin, and the bundle build fails on them.

## The forwarder

When a Go command has shipped in one bundle, the next bundle turns its Python `alpha` entry into a forwarder. The first callers of the Go command are then the ones that chose its final name.

1. The forwarder runs only when `TBHBDEV_PATH` is set and `TBHBDEV_FORWARDED` isn't. Otherwise it runs the Python code, so Python started outside a bundle still works.
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
| Briefs and prompts that roles write | agent-config | any | The agent-config maintainer, one pull request per wave |
| Repository docs | `AGENTS.md`, runbooks, and designs in each repository, such as this wiki's `AGENTS.md` naming `tbhbdev alpha install` | any | Each repository's maintainer, one pull request per wave |
| Scripts | Operator scripts in `~/.local/libexec/tbhb` and the ops sync scripts | `version-check`, `gh-app`, `agent` | The ops maintainer |

For each command, in order:

1. **Ship.** The Go command ships in a bundle, and the fleet check shows every host on that bundle or later. The fleet check is `tbhbdev alpha version-check` (existing) until wave 4 ports it as `tbhbdev install check`.
2. **Forward.** The next bundle makes the Python `alpha` entry a forwarder.
3. **Move.** Each caller's owner changes the call to the final name. `rg 'tbhbdev alpha <command>'` across tools, tbhbdev, agent-config, ops, and wiki finds the work.
4. **Hold.** A planned lint in tbhb-dev/rules, fed the ported names from `names.toml`, fails on new `tbhbdev alpha <command>` text in any repository that runs the rules checks.
5. **Remove.** When every host's forwarder log shows no call for that command for 14 days, the Python code for it goes. `tbhbdev alpha <command>` then exits 2 with a usage message that names the final command.

Steps 3 and 4 run per wave, not per command, so each caller repository takes one pull request per wave.
