---
type: Design Appendix
title: Agent roles alternatives
description: The options the agent roles design didn't take, and why.
tags: [agent-roles, alternatives, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Appendix B. Alternatives considered

Back to [the design](README.md).

## Where the registry lives

| Option | Why not |
| --- | --- |
| One file in the public rules repository, synced to every repository by the baseline | Entries name private repositories and host classes, which can't appear in a public repository. Every repository would also carry every role, though each needs only its own outputs |
| A public core in rules plus a private overlay | Workable, and the baseline already has visibility variants. It splits one entry across two repositories and two reviews, and a public core holds little once private scopes leave it. Kept as the fallback |
| The private IT service management records repository, beside the Requests whose role labels it feeds | The role label set is one output of five. The other four, launch definitions above all, are harness configuration, which the operator gave to the harness configuration owner |
| The private harness configuration repository | Taken. It already holds account routing and harness settings, and the operator made its owner this design's owner |

## Registry format

| Option | Why not |
| --- | --- |
| CUE, as the content kernel uses for templates | The operator chose CUE for content templates and constraints, not configuration. TOML matches the routing file beside it. A CUE schema can still validate the TOML later |
| YAML | No other file in the repository uses it, and its typing surprises are a known source of drift |
| One file per role | Diffs that move authority between roles would span files, which makes a widening harder to see in review |

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

## Keeping sessions as the unit

Each long-running session could keep its own prompt and settings, with the registry listing only names. That keeps today's drift, since the session's habits stay the real definition. It also doesn't serve the operator's direction that workflows launch agents per Request.
