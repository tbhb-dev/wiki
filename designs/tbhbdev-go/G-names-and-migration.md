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

Go has no `alpha` (D5). Each command takes its final name when it ports, and `tbhbdev alpha <command>` keeps working through the Python fallthrough until then. The commands below come from the `alpha` usage text at tools `938f65f` (existing, a local clone older than the outline's basis). A command added since then gets a row when it ports.

## The name map

One file in tools, `names.toml`, holds this map with the first bundle version that carries each final name (new). The installer, the forwarders, and the caller check read it, and this table mirrors it.

| Today | Final name | Plugin binary | Wave |
| --- | --- | --- | --- |
| `alpha agent exec`, `wait`, `status`, `kill`, `clean`, `keep` | `agent exec`, `wait`, `status`, `kill`, `clean`, `keep` | `tbhbdev-agent` | 1 |
| `alpha agent ssh-serve` | `agent ssh-serve` | `tbhbdev-agent` | 1 |
| `alpha agent-env` | `agent env` | `tbhbdev-agent` | 2 |
| `alpha gh-shim` | `agent gh-shim` | `tbhbdev-agent` | 2 |
| `alpha identity record`, `trailer`, `consume`, `check`, `stamp-docs` | `identity record`, `trailer`, `consume`, `check`, `stamp-docs` | `tbhbdev-identity` | 2 |
| `alpha verify-range` | `identity verify-range` | `tbhbdev-identity` | 3 |
| `alpha guard-gh-pr` | `hook gh-pr` | `tbhbdev-hook` | 2 |
| `alpha guard-op` | `hook op` | `tbhbdev-hook` | 2 |
| `alpha state-guard` | `hook state` | `tbhbdev-hook` | 2 |
| `alpha session-context` | `hook session-context` | `tbhbdev-hook` | 2 |
| `alpha guard-gh-poll` | `hook gh-poll` | `tbhbdev-hook` | 2 |
| `alpha guard-gh-config-dir` | `hook gh-config-dir` | `tbhbdev-hook` | 2 |
| `alpha guard-scripts` | `hook scripts` | `tbhbdev-hook` | 2 |
| `alpha pre-push-guard` | `hook pre-push` | `tbhbdev-hook` | 3 |
| `alpha pr check`, `create`, `edit`, `hook`, `backfill-opened`, `rerun`, `delete-branch`, `close` | `pr check`, `create`, `edit`, `hook`, `backfill-opened`, `rerun`, `delete-branch`, `close` | `tbhbdev-pr` | 2 |
| `alpha merge` | `pr merge` | `tbhbdev-pr` | 2 |
| `alpha review draft`, `check` | `pr review draft`, `check` | `tbhbdev-pr` | 2 |
| `alpha pr-stats` | `pr stats` | `tbhbdev-pr` | 3 |
| `alpha gh-app` | `gh-app` | `tbhbdev-gh-app` | 2 |
| `alpha repo bootstrap` | `repo bootstrap` | `tbhbdev-repo` | 2 |
| `alpha loop` | `loop` | `tbhbdev-loop` | 2 |
| `alpha issue close` | `issue close` | `tbhbdev-issue` | 3 |
| `alpha brief new`, `check` | `brief new`, `check` | `tbhbdev-brief` | 3 |
| `alpha install` | `install` | `tbhbdev-install` | 3 |
| `alpha version-check` | `install check` | `tbhbdev-install` | 3 |

The top-level names `admin`, `secret`, `adr`, `agy`, `cue`, and `issue` keep their names and port in wave 3 ([appendix H](H-waves.md)). `analytics` and `host` follow D2.

The harness hooks share one binary, `tbhbdev-hook`, because they run on every tool call and a small binary starts fastest. The pull request commands share `tbhbdev-pr` because they share the GitHub adapters.

## Collisions with Python names

A Go plugin takes a name before Python does. Three rules keep that from hiding a Python command:

- **Built-in Python names.** A Go plugin for `admin`, `secret`, `issue`, or another top-level Python name ships only when it covers every Python subcommand of that name. `tbhbdev-issue` therefore ports `import` and `close` together. A bundle test compares the Go plugin's subcommands with the Python ones from the same commit.
- **Entry-point plugins from other repositories.** graph and the private ITSM repository add names through `tbhbdev.plugins`. The bundle can't see those, so `tbhbdev install check` lists each host's entry-point names and reports any that a bundle plugin shadows.
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
| Harness hooks | The user-level harness settings that `tbhbdev alpha install` writes | the guards, `state-guard`, `session-context`, `identity` | The installer reads `names.toml` and writes final names for every command its bundle carries |
| Git hooks | Each repository's hooks, installed by `tbhbdev alpha install` | `identity`, `pre-push-guard` | The same installer step |
| The forced command | Each host's authorized keys | `alpha agent ssh-serve` | The ops maintainer, host by host, in phase 3 ([appendix I](I-protocol.md#the-transition)) |
| Brief templates | `tbhbdev alpha brief new` in tools | `alpha pr check`, `alpha merge`, `alpha agent exec` | The tools maintainer, in the change after the forwarder ships |
| Briefs and prompts that roles write | agent-config | any | The agent-config maintainer, one pull request per wave |
| Repository docs | `AGENTS.md`, runbooks, and designs in each repository, such as this wiki's `AGENTS.md` naming `tbhbdev alpha install` | any | Each repository's maintainer, one pull request per wave |
| Scripts | Operator scripts in `~/.local/libexec/tbhb` and the ops sync scripts | `version-check`, `gh-app`, `agent` | The ops maintainer |

For each command, in order:

1. **Ship.** The Go command ships in a bundle, and `tbhbdev install check` shows every host on that bundle or later.
2. **Forward.** The next bundle makes the Python `alpha` entry a forwarder.
3. **Move.** Each caller's owner changes the call to the final name. `rg 'tbhbdev alpha <command>'` across tools, agent-config, ops, and wiki finds the work.
4. **Hold.** A planned lint in tbhb-dev/rules, fed the ported names from `names.toml`, fails on new `tbhbdev alpha <command>` text in any repository that runs the rules checks.
5. **Remove.** When every host's forwarder log shows no call for that command for 14 days, the Python code for it goes. `tbhbdev alpha <command>` then exits 2 with a usage message that names the final command.

Steps 3 and 4 run per wave, not per command, so each caller repository takes one pull request per wave.
