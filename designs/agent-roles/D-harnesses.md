---
type: Design Appendix
title: Agent roles harness definitions
description: How a role renders into Claude Code, Codex, and Antigravity definitions, how each combines with account settings, and how the launcher's flags map to roles.
tags: [agent-roles, harness, launcher, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Appendix D. Harness definitions

Back to [the design](README.md).

Each harness fact here cites [appendix A](A-evidence.md#harness-documentation). Local versions were Claude Code 2.1.330, codex-cli 0.171.0, and Antigravity 1.5.14. The renderers and the delivery are planned.

## Terms

- **Launcher profile:** `agent exec --profile build|review`. It sets the checkout, credentials, and permission mode of a run.
- **Claude profile:** `agent exec --claude-profile`, which picks the Claude account a run signs in with.
- **Codex config profile:** a file `$CODEX_HOME/<name>.config.toml`, selected by `codex -p <name>`.
- **Run class:** `agent exec --class`, a label the run index records. It's unrelated to the review classes N, M, R, D, and K in the harness configuration repository's review policy.
- **Run role:** `agent exec --role`, also recorded. Only `reviewer` has behavior today: the review-launcher check.

## Claude Code

### Fields

An agent definition is a Markdown file with YAML frontmatter, and the body is the system prompt. `name` and `description` are required. The documented fields:

| Field | What it does | Use for roles |
| --- | --- | --- |
| `tools` | Allow list of tools. Omitted, the agent inherits every tool | Narrowing. Whether an entry with a Bash specifier narrows Bash to that pattern is unverified |
| `disallowedTools` | Tools removed from the inherited or listed set. An entry with a specifier, such as `Bash(git push *)`, removes the whole tool | Narrowing, whole tools only |
| `model` | `sonnet`, `opus`, `haiku`, `fable`, a full model ID, or `inherit` | Launch default |
| `effort` | `low` to `max`. Overrides the session's effort, not the environment variable | Launch default |
| `permissionMode` | `default`, `acceptEdits`, `auto`, `dontAsk`, `bypassPermissions`, `plan` | Not emitted |
| `mcpServers` | Server names already configured, or inline definitions | Not emitted |
| `hooks` | Lifecycle hooks scoped to the agent. All events are allowed, and `Stop` becomes `SubagentStop` | Narrowing, from the fixed guard list only |
| `skills` | Skills preloaded in full | Allowed |
| `memory` | `user`, `project`, or `local` persistent memory | Not emitted. State lives in records |
| `maxTurns` | Turn limit, after which output is marked partial | Allowed |
| `isolation` | `worktree` runs in a temporary worktree | Not emitted. The launcher makes worktrees |
| Others | `background`, `omitClaudeMd`, `initialPrompt`, `color`, `autoCompactWindow`, `experimental` | Not emitted |

### How a definition combines with account settings

- **Denies add, and a definition can't lift them.** Rules evaluate deny, then ask, then allow, and an allow can't carve an exception out of a deny. Deny rules block in every mode, including `bypassPermissions`. Managed rules can't be overridden at all. A definition's `tools` list selects from what's available and can't restore a tool a settings deny removed. That last point follows from the documented order, and the docs don't state it in those words, so it's unverified until step 1.
- **Command-level limits need settings or hooks.** Because `disallowedTools: Bash(git push *)` removes all of Bash, a role that keeps Bash but can't push needs a `permissions.deny` rule or a `PreToolUse` hook that denies.
- **Hooks add.** Hook entries merge across levels, and a definition's hooks run while that agent runs.
- **Three fields can widen.** `permissionMode` takes effect when the parent is in `default` or `dontAsk`, so `acceptEdits` there approves edits the account would deny by prompt. `mcpServers` brings in servers the account didn't configure. A hook can return `allow` and approve a call. The generator never emits the first two, and emits hooks only from the fixed guard list, whose members deny or pass and never allow.
- **Plugin definitions** ignore `hooks`, `mcpServers`, and `permissionMode`, so roles aren't shipped as plugins.

### Under the launcher today

| Launcher setting | Effect on a role definition | Evidence |
| --- | --- | --- |
| `bypassPermissions` for full-access runs | The definition's `permissionMode` is ignored. Account denies still block | Existing |
| Generated settings with `disableAllHooks: true` | Account hooks are off. Whether frontmatter hooks are off too is unverified | Unverified |
| `--setting-sources user` | Account settings and user-level agents load. Whether project `.claude/agents/` loads is unverified | Unverified |
| `claude -p` in a fresh run clone | Project subagent frontmatter hooks and inline `mcpServers` aren't used in a folder never trusted | Existing |
| `--strict-mcp-config` | Only servers passed for the run load | Existing |

The design therefore delivers definitions at user level, as `<role>.md` in the `.claude/agents/` folder of the account's home, installed by the deployer, and passes the run's role with `--agents <file>` and `--agent <role>`. `--agents` outranks project and user agents, and only managed definitions outrank it. With `--agent`, the main thread takes the definition's tool restrictions and model. Whether it also takes the definition's hooks is unverified. For role hooks to bind, the launcher stops setting `disableAllHooks` for runs that carry a role. Removing that setting makes runs stricter, so it is the tools owner's engineering call under R13.

### Example

```markdown
---
name: reviewer
description: Launches reviewer pairs for green heads, with the context of the area repository.
model: sonnet
effort: medium
disallowedTools: Edit, Write
hooks:
  PreToolUse:
    - matcher: Bash
      hooks:
        - type: command
          command: tbhbdev alpha hook guard-role --role reviewer
---

You are the reviewer kind. ...
```

The `guard-role` hook is planned. It denies `git push`, `tbhbdev alpha merge`, and the approval command for this role, which a `disallowedTools` entry can't do without removing Bash.

## Codex

- **No definition for the main thread.** `codex agents` only browses sessions. Custom agent files exist in the `.codex/agents/` folder of the home directory or of a project, with `name`, `description`, `developer_instructions`, and model settings. They serve only subagents a session spawns, and no documented flag starts a session as one.
- **The role's launch is a config profile.** The renderer writes `<role>.config.toml` with `developer_instructions`, `model`, `model_reasoning_effort`, and `features.*` switches that turn off tools the role doesn't use. The deployer writes it to a root-owned path, and the launcher copies it into the run's private `CODEX_HOME`, failing closed on a missing or mismatched file, and passes `-p <role>`. `model_instructions_file` replaces the built-in instructions, so it isn't used. A project `.codex/config.toml` can't select a profile, and the launcher marks the run's project untrusted anyway.
- **Guardrails are rules files.** A decision record of the harness configuration repository has each run get the account's rules as a copy in its `CODEX_HOME/rules/`, and a live probe showed such a copy binds a run. The design doesn't rely on that floor until a check on the run host verifies it. The role adds its own `<role>.rules` beside it, so role rules add to the floor and can't remove it.
- **Agent files can't loosen.** `sandbox_mode`, `approval_policy`, `model_provider`, and `mcp_servers` in an agent file don't override the parent, and subagents inherit the parent's sandbox and approval.
- **Unverified:** whether the launcher's `-c` overrides beat a profile file's values. The launcher passes `-m` and `-c model_reasoning_effort` today, so step 1 tests the precedence before step 7 drops them. Also unverified: whether a profile file can carry `[hooks]`, and whether `multi_agent_v2 = false` limits custom agents on 0.171.0.

## Antigravity

- **Custom agents** are Markdown files with YAML frontmatter, the body as system prompt, at `.agents/agents/<name>.md` or under `~/.gemini/config/agents/`. `--agent <name>` selects one as the main agent, also in headless `-p` runs.
- **Fields** include `tools`, `model` (`inherit`, `flash`, or `pro`), `mainAgent`, `subagent`, `commandExecutionPolicy` (`off`, `auto`, `eager`, or `sandbox`), `mcpServers`, `skills`, `plugins`, `rules`, `agents`, `hooks`, `hidden`, `inheritMcp`, and `excludeDefaultComponents`.
- **Combination with settings:** subagents inherit their parent's safety configuration. Whether a main-thread custom agent's `commandExecutionPolicy` or `tools` can loosen a settings deny, and how either interacts with the launcher's `--dangerously-skip-permissions`, are unverified. Until step 1 settles them, the renderer emits only `tools`, `model`, and guard `hooks`, and Antigravity roles stay on review and plan-mode work, as the roster already keeps them.

## Capability bundles per harness

A capability bundle has five fields, and the launcher composes the kind's base with the injected capabilities into one definition per run. The composition is new, from the operator's direction (R31). Each field keeps the limits above: deny rules add and a definition can't lift them, an allow never lifts an account deny, and no emitted field widens authority.

| Bundle field | Claude Code | Codex | Antigravity |
| --- | --- | --- | --- |
| Allow rules | `tools` in the run's definition, and `permissions.allow` in the run's generated settings. Allow rules have no effect under `bypassPermissions` | Not emitted, because a Codex rule that allows skips approval | `tools` in the run's definition |
| Deny rules | `permissions.deny` in the run's generated settings for command patterns, and `disallowedTools` for whole tools | Forbid rules in the run's `<role>.rules` file | Not emitted until step 1 shows how Antigravity combines them |
| Hooks | Frontmatter `hooks`, from the guard list. Whether they bind under the launcher is unverified | Unverified whether a profile file can carry `[hooks]`, so not emitted yet | Frontmatter `hooks`, from the guard list |
| Credentials | Not in the definition. The launcher mints them per run, scoped to the area repository and the capability, as it scopes the build run's token today | Same | Same |
| Context fragments | Appended to the definition's body, the system prompt | Appended to `developer_instructions` in the run's profile file | Appended to the definition's body |

A credential scope never widens authority. It narrows what the account could reach to the one repository and the actions the capability names.

## What the generator may emit

The rule is that no emitted field widens authority. `skills` and `maxTurns` add context or a turn limit and grant nothing, so they are emitted. The excluded fields are the ones that can widen, plus a few the launcher or the records already cover.

| Harness | Emitted | Never emitted |
| --- | --- | --- |
| Claude Code | `name`, `description`, `tools`, `disallowedTools`, `model`, `effort`, `skills`, `maxTurns`, guard `hooks` | `permissionMode`, `mcpServers`, `memory`, `isolation`, any hook outside the guard list |
| Codex | `developer_instructions`, `model`, `model_reasoning_effort`, `features.*` set to false, a role rules file | `sandbox_mode`, `approval_policy`, `mcp_servers`, `model_instructions_file`, any rule that allows |
| Antigravity | `name`, `description`, `tools`, `model`, guard `hooks` | `commandExecutionPolicy`, `mcpServers`, `plugins` |

## Launcher flags and roles

| Flag today | What it does today | With a role |
| --- | --- | --- |
| `--profile build\|review` | Sets checkout, credentials, and permission mode | Defaults from the kind's `launch.profile`. Reviewer runs are always `review` |
| `--role` | Recorded. `reviewer` triggers the review-launcher check | Names the kind. The review check reads the kind instead of a pane title |
| `--class`, `--variant` | Recorded labels | Unchanged. Workflows may set them for telemetry |
| `--harness`, `--model`, `--effort` | Required or defaulted per run | Default from the role. A caller's value still wins, and the run records both |
| `--claude-profile`, `--account`, `--ops` | Pick the Claude account and the host account | Unchanged. The registry doesn't hold credentials |

A long-running session today is an interactive harness started by hand in a terminal pane, named by its title. In the interim it starts as `claude --agent <role>` from the same user-level definition a launch uses. The launcher's `start` run kind exists in code with no command-line verb, so whether it can host such sessions is unverified.
