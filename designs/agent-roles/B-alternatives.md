---
type: Design Appendix
title: Agent roles alternatives
description: The options the agent roles design didn't take, and why.
tags: [agent-roles, alternatives, design]
status: draft
parent: README.md
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-opus-5-5
    effort: medium
    surface: cli
    session: sha-256-64:e48956ec8ffa792a
    subagent: sha-256-64:63469a419533009b
    at: "2026-11-22T23:16:43Z"
  - role: assisted
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:e48956ec8ffa792a
    at: "2026-11-22T23:16:43Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Appendix B. Alternatives considered

Back to [the design](README.md).

## Where the registry lives

| Option | Why not |
| --- | --- |
| One file in the public rules repository, synced to every repository by the baseline | The registry feeds harness configuration, which the operator gave to the harness configuration owner, and every repository would carry a file only the launcher reads |
| The private IT service management records repository, beside the Requests whose `area:` and `cap:` labels it feeds | The Request label set is one output of five. The other four, launch definitions above all, are harness configuration, which the operator gave to the harness configuration owner |
| The private harness configuration repository for the registry data and harness parts, with a public page generated from it | Taken, by the operator's decision (D8) |

## Registry format

| Option | Why not |
| --- | --- |
| CUE, as the content kernel uses for templates | The operator chose CUE for content templates and constraints, not configuration. TOML matches the routing file beside it. A CUE schema can still validate the TOML later |
| YAML | No other file in the repository uses it, and its typing surprises are a known source of drift |
| One file per kind | Diffs that move a capability between kinds would span files, which makes a widening harder to see in review |

## Claude Code delivery

| Option | Why not |
| --- | --- |
| Project `.claude/agents/<role>.md` in each repository | A run can write its own clone, so a guardrail there isn't a floor. Project frontmatter hooks don't run in an untrusted `claude -p` folder, and project loading under `--setting-sources user` is unverified |
| A plugin that ships every role | Plugin agents ignore `hooks`, `mcpServers`, and `permissionMode`, so a plugin can't carry hook guardrails |
| Managed settings definitions | They outrank everything, which suits a floor, but managed settings are the operator account's machinery today, and a role per account doesn't fit them |
| `--append-system-prompt` with the role's text only | It carries instructions but no tool limits or hooks |

## Codex delivery

| Option | Why not |
| --- | --- |
| Custom agent files as the main thread | No documented flag starts a session as a custom agent |
| `model_instructions_file` | It replaces Codex's built-in instructions, not adds to them |
| A prepended block in the brief only | It carries instructions but no model, features, or rules. It stays the fallback if profile files fail step 1 |

## Per-repository owner assignments

An earlier form of the answer to D1 kept owner assignments per repository, recorded by the coordinator. The operator superseded it (R30): an agent looks up the context each task needs, so ownership adds nothing for agents, and assignments would grow with every repository.

## Keeping sessions as the unit

Each long-running session could keep its own prompt and settings, with the registry listing only names. That keeps today's drift, since the session's habits stay the real definition. It also doesn't serve the operator's direction that workflows launch agents per Request.
