---
type: Design Appendix
title: Agent roles evidence
description: The sources behind each claim in the agent roles design, labeled existing, new, or unverified.
tags: [agent-roles, evidence, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore unsandboxed sessionless cutover -->

# Appendix A. Evidence

Back to [the design](README.md).

Every read here was taken on 2026-11-22. A claim marked existing was read from the named source at the named commit. A claim marked unverified has no source that settles it yet.

## Public repositories

### tools, at `b8172690`

- **Existing.** `src/tbhbdev/alpha/agent/__init__.py` defines `agent exec` with `--harness codex|claude|agy` (default `codex`), required `--model`, `--effort`, and `--profile build|review`, and optional `--role`, `--class`, `--variant`, `--claude-profile`, `--account` or `--ops`, and `--unsandboxed`.
- **Existing.** `src/tbhbdev/alpha/agent/decisions/launcher.py`: a run counts as reviewer-role when `role == "reviewer"` or the profile is `review`. It may start only from a session on the launcher list, never from the session that owns the repository. The built-in list names three review sessions with any repository. The module's docstring calls the list a bump against honest mistakes, not a barrier against a session that forges its name.
- **Existing.** `docs/reference/alpha/agent-configuration.md`: the launcher reads the launching session's name from its terminal pane title. `agent-review-launchers.json` in the state directory adds launchers and owners. Every run has the agent account's full access, and the launcher offers no sandboxed mode. Claude runs load user settings and the run's generated settings, ignore project settings, and turn off hooks and the sandbox.
- **Existing.** `src/tbhbdev/alpha/agent/run_host.py`, `index_entry`: `class`, `variant`, and `role` are copied into the run index. No other code reads `class` or `variant`, so they're recorded labels.
- **Existing.** `src/tbhbdev/alpha/agent/decisions/policy.py`: a full-access Claude run's generated settings are `{"disableAllHooks": true, "sandbox": {"enabled": false}}`.
- **Existing.** `src/tbhbdev/alpha/agent/decisions/argv.py`: a Claude run gets `--model`, `--effort`, `--permission-mode` (`bypassPermissions` for unsandboxed runs, otherwise `auto` for build and `dontAsk` for review), `--settings <generated>`, `--setting-sources user`, and `--strict-mcp-config`. A Codex run gets `codex exec --strict-config -C <worktree>`, its project marked untrusted, and `-m` and `-c model_reasoning_effort` overrides. An Antigravity run gets `--model`, `--dangerously-skip-permissions` when unsandboxed, and `--mode plan` for review.
- **Existing.** `docs/reference/alpha/loop-configuration.md`: `tbhbdev alpha loop` takes an `approval_command`, spawn caps, and per-rule modes `off`, `shadow`, or `on`. The runner rejects every `on` mode, and the command exits 1 because live intake isn't wired, so it executes no approval today.
- **Existing.** `src/tbhbdev/alpha/agent/model.py` defines run kinds `exec` and `start`. Only `exec` has a command-line verb. What starts a `start` run is unverified.
- **Existing.** `docs/reference/alpha/no-self-approval.md`: approval comes only from `tbhb-approve REPO N SHA`, which a guard denies when the caller's ledger lists that pull request. Review runs post a `COMMENT`. The guard is not a security boundary.
- **Existing.** `docs/reference/alpha/review-drafts.md`: a review draft can't hold `APPROVE`, so a reviewer's GO is a `COMMENT`.

### wiki, at `a4a81ac`

- **Existing.** `.github/CODEOWNERS` is one owner-less `*` line. `.github/rulesets/main.json` requires no approving review and no code owner review.
- **Existing.** `models/README.md` sets the review pair by the model that wrote the diff, and Sonnet as the default Claude session model.
- **Existing.** [wiki#76](https://github.com/tbhb-dev/wiki/pull/76), open at `b890ae1`, describes a Handler field with two values, Operator and Agent. D7 in [decisions.md](decisions.md) keeps that field and adds a `role:<name>` label on agent Requests.
- **Existing.** `designs/event-gateway/README.md` lists, as known debt, that no durable role name exists for harness configuration. This design names that role the harness configuration owner.

### rules, at `dda445c`

- **Existing.** `repo/baseline/README.md` and `repo/baseline/files/.github/CODEOWNERS`: the baseline holds a shared owner-less CODEOWNERS file, with visibility variants, and a writer that substitutes placeholders.

## The private harness configuration repository

The repository is private, so this design names it by role and gives no paths from it. Reviewers who can read it get the paths and the commit from the design's author.

- **Existing.** Its account routing file grants the ops account two playbook capabilities, a check and an apply, with conditions. The agent account holds no capability and is the routing default. A decision record holds the operator's answer granting the two and holding a third.
- **Existing.** A decision record lets the automated review approve one class of permission change to that repository without the operator, while a wider class still goes to the operator. Those review classes belong to that repository's review policy and are unrelated to the launcher's `--class` label.
- **Existing.** A decision record says each Codex run has a private `CODEX_HOME` holding only its sign-in file. A rules file copied into `$CODEX_HOME/rules/` refused a forced push in a live run on codex-cli 0.166.0.
- **Existing.** The agent account's Claude Code settings carry deny rules, the approval command among them. The design relies on no account deny it hasn't verified on the host that runs the role.

## Harness documentation

### Claude Code

Local version: `claude --version` printed `2.1.330 (Claude Code)`. The documentation pages carry no version, so each claim is the page as read on 2026-11-22.

- **Existing.** [Subagents](https://code.claude.com/docs/en/sub-agents.md): the frontmatter fields, the rules for `tools` and `disallowedTools`, that a `disallowedTools` entry with a specifier removes the whole tool, that `permissionMode` is ignored when the main conversation is in `bypassPermissions`, `acceptEdits`, or auto mode, that plugin subagents ignore `hooks`, `mcpServers`, and `permissionMode`, the location priority, and `--agents` and `--agent`.
- **Existing.** [Permissions](https://code.claude.com/docs/en/permissions.md): rules evaluate deny, then ask, then allow, and an allow can't carve an exception out of a deny. Project subagent frontmatter hooks and inline `mcpServers` aren't used in a `claude -p` run in a folder never trusted. Managed rules can't be overridden by any other level.
- **Existing.** [Permission modes](https://code.claude.com/docs/en/permission-modes.md): deny rules block in every mode, including `bypassPermissions`, and allow rules have no effect there.
- **Existing.** [Hooks](https://code.claude.com/docs/en/hooks.md): hook entries merge across settings levels. Subagent frontmatter hooks run only while that subagent runs, and `Stop` becomes `SubagentStop`. **Unverified:** whether `disableAllHooks` also turns off frontmatter hooks, since the page read doesn't say.
- **Existing.** [CLI reference](https://code.claude.com/docs/en/cli-reference.md): `--agents` takes JSON, or a file path with `--print` from v2.1.281. `--agent` sets the session's agent. `--setting-sources` lists `user`, `project`, and `local`. **Unverified:** whether `--setting-sources user` stops project subagents in `.claude/agents/` from loading.

### Codex

Local version: `codex --version` printed `codex-cli 0.171.0`.

- **Existing.** `codex --help`: `-p, --profile <CONFIG_PROFILE_V2>` layers `$CODEX_HOME/<name>.config.toml` on the base user config. `codex agents` opens a command center to browse and switch sessions, and defines nothing.
- **Existing.** `codex features list`: `multi_agent` is deprecated and true, and `multi_agent_v2` is stable and false. The subagents page says current releases enable subagent workflows by default. **Unverified:** how the two relate on 0.171.0.
- **Existing.** [Configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference), redirected from developers.openai.com and carrying no version: `developer_instructions` adds developer instructions to the session. `model_instructions_file` replaces the built-in instructions. `instructions` is reserved. `agents.<name>.description` and `agents.<name>.config_file` declare a role with its own config layer. A project `.codex/config.toml` can't set `profile` or `profiles`. `hooks` and `features.hooks` configure lifecycle hooks.
- **Existing.** [Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents.md), no version: standalone agent files in the `.codex/agents/` folder of the home directory or of a project need `name`, `description`, and `developer_instructions`. `sandbox_mode`, `approval_policy`, `model_provider`, and `mcp_servers` in an agent file don't override the parent, and subagents inherit the parent's sandbox and approval settings. The page names no flag that starts a session as a custom agent.

### Antigravity

Local version: `agy --version` printed `1.5.14`.

- **Existing.** `agy --help`: `--agent` selects the agent for the session, `--settings` takes session settings, `--mode` takes `accept-edits`, `auto`, or `plan`, and `agy agents` lists agents.
- **Existing.** `agy changelog`, read locally at 1.5.14: Markdown custom agents with YAML frontmatter and the body as system prompt. Frontmatter fields include `mainAgent`, `subagent`, `hidden`, `inheritMcp`, `commandExecutionPolicy`, `model`, `tools`, `agents`, `skills`, `plugins`, `rules`, `hooks`, and `excludeDefaultComponents`. Project custom agents in `.agents/agents/` work with `--agent` in headless runs, and `--agent` with an unknown name exits 1 in headless runs.
- **Existing.** [Custom subagents](https://antigravity.google/docs/subagents.md), no version: workspace agents at `.agents/agents/<name>.md` and global agents under `~/.gemini/config/agents/`. `commandExecutionPolicy` defaults to `sandbox`, and subagents inherit safety configuration from their parent. **Unverified:** whether a custom agent can loosen a settings deny.

## Operator and coordinator records

These records sit in the operator's private decision notes, which aren't a repository. Each is cited by date and topic, and labeled by who decided. The design states them in plain words.

| Key | Date | Who decided | What it says |
| --- | --- | --- | --- |
| R0 | 2026-11-11, narrowed 2026-11-15 | Operator | The constitution: reserved matters, the operator's stop, never approving your own change, and that technical restrictions on agent accounts are engineering choices |
| R1 | 2026-11-07 | Operator | No approving your own pull request, enforced by hooks |
| R2 | 2026-11-09 | Operator | The approving session may review and approve under the coordinator's rules, never a pull request it or the coordinator opened |
| R3 | 2026-11-11 | Operator, shape by the coordinator | Remove the operator's lines from CODEOWNERS everywhere. Each repository keeps one owner-less `*` line |
| R4 | 2026-11-12 | Operator | Designs name roles and event-triggered workflows, never sessions |
| R5 | 2026-11-12 | Operator grant, limits by the coordinator | The admin App, with refusals enforced in the tool that mints its tokens. It never approves pull requests |
| R6 | 2026-11-12 | Operator | Admin console work goes through the admin role's virtual machine only |
| R7 | 2026-11-12 | Operator | The coordinator decides 1Password Connect grants as needed |
| R8 | 2026-11-13 | Operator | The coordinator improves its own workings, without loosening checks that review it |
| R9 | 2026-11-15 | Coordinator's call, then the operator's direction | Owners send green notices straight to the approver. Each owner runs its own merge queue, except the approval |
| R10 | 2026-11-15 | Operator | Repository and lane owners decide their own work. The coordinator steers priorities and isn't an approval step |
| R11 | 2026-11-15 and 2026-11-22 | Operator | Owners deploy their own merged services, and may log in as the ops account to do it |
| R12 | 2026-11-15 and 2026-11-22 | Operator, mechanics by the coordinator | The standup of the coordinator's sub-coordinators, now every 2 hours, from a participants file |
| R13 | 2026-11-15 | Operator | The Laws narrowed to exposure and to authority over decisions |
| R14 | 2026-11-22 | Operator, with the coordinator's calls | Workflows behind an event gateway, no coordination sessions in the end state, and new repository owners named when each repository is created |
| R15 | 2026-11-22 | Operator, with the coordinator's calls | Agent runs push their own branches and are autonomous |
| R16 | 2026-11-22 | Operator, mechanics proposed by the coordinator | Requests split by who picks them up, operator or agent |
| R17 | 2026-11-22 | Operator | Who approves Normal Changes |
| R18 | 2026-11-22 | Operator | Authors allowed for decision auto-merge are listed in CODEOWNERS |
| R19 | 2026-11-22 | Operator | Asks between roles move onto Requests, routed by event to a run |
| R20 | 2026-11-22 | Operator | Handler field alone, Operator plus one value per agent role. After the operator's feedback on D1 of this design, the design keeps the field at Operator and Agent and adds `role:<name>` labels, because the field limits its values |
| R21 | 2026-11-22 | Operator | This design, its owner, and what the registry feeds |
| R22 | 2026-11-22 | Operator, with the coordinator's reading | No limit of one pull request in review per repository. That review launchers start pairs for every green head at once is the coordinator's reading |
| R23 | 2026-11-22 | Coordinator's calls | Reviews post on GitHub through the reviews App, as `COMMENT` only for now |
| R24 | 2026-11-08 | Operator | A deterministic loop that spawns independent agents per event replaces the long-lived coordination session |
| R25 | 2026-11-12 | Operator | Answers on the review and merge worker: it runs as a service account with no harness, and it may approve, with no model read, a rebuilt replacement whose tree equals an approved one |
| R26 | 2026-11-06 | Operator | The wiki repositories merge on green, without an approval |

### Two records that conflict

**Who approves pull requests.** R2 lets the approving session approve under the coordinator's rules, and tools' `review-drafts.md` says the coordinator runs the approval command. R10 lists approvals only through the approving session among the things it leaves unchanged. The design records the narrower reading: pull request approvals go through the approver. Whether the coordinator may still approve one is unverified.

**Who starts reviews.**

R9 says each owner's merge queue starts the reviews on each new head. The launcher refuses a reviewer-role run started from the session that owns the repository (tools, `launcher.py`). The code is what runs today, so the design treats starting reviews as a request from the owner to a review launcher. Whether R9 meant anything else is unverified.
