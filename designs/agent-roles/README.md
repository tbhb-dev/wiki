---
type: Design Draft
title: Agent roles
description: One registry of tbhb-dev's agent roles, their authority today, and how each role becomes a launch in Claude Code, Codex, and Antigravity.
tags: [agent-roles, authority, launcher, harness, design]
status: draft
owner: Harness configuration owner
updated: "2026-11-22"
decisions_open: 1
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore unsandboxed sessionless cutover -->

# Agent roles

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Harness configuration owner | 2026-11-22 | 1 open decision in [decisions.md](decisions.md), about 3 minutes |

## Summary

tbhb-dev's roles live in many places today: decision records, the launcher's code, a review-launcher file, a standup participants file, rulesets, and the habits of long-running sessions. Each copy drifts on its own, and none of them is a definition a workflow can launch.

This design writes today's roles into one machine-readable registry and widens no one's authority. The registry holds role kinds and their capabilities, and nothing per repository. It generates the `area:` and `cap:` labels on agent Requests, the review-launcher check, the standup participants, and one launch definition per kind for each harness, and CODEOWNERS stays checked as is.

For the operator, the change is that a role kind becomes a file that can be read and reviewed. A workflow can then launch a run of that kind for each Request, with the context of the repository the Request names, which is the sessionless direction the operator set.

## Proposal

When this is built, these pieces exist:

- **The registry**, `roles/registry.toml` in the private harness configuration repository, owned by the harness configuration owner. By the operator's answer to D1, it defines role kinds (builder, reviewer, approver, coordinator, and admin) and their capabilities only, with no role assigned to a repository. Each kind says what it decides, may merge or approve, may never do, which capabilities it holds, and its launch defaults. Only the operator creates or widens a kind or a capability. [Appendix C](C-registry.md) gives the schema and today's kinds.
- **A registry check**, a Bazel test in that repository. It fails when an authority line has no citation, when any table names a repository or host, or when a kind or capability is added or widened and the change carries no operator record.
- **Generators**, commands in tools. They write the generated files and check the ones that stay hand-kept, one output each ([How it works](#generated-and-checked-outputs)).
- **Harness definitions**, rendered per kind: a Claude Code agent definition, a Codex profile file with its rules file, and an Antigravity custom agent. The harness configuration deployer installs the Claude Code and Antigravity definitions at user level on every agent host, and writes the Codex files to a root-owned path that the launcher copies into each run's private `CODEX_HOME`. [Appendix D](D-harnesses.md) maps the fields.
- **Launcher support** for `tbhbdev alpha agent exec --role <name>`. The launcher reads the role's defaults and passes the role's definition to the harness. The flag already exists and is recorded, but only `reviewer` changes behavior today (appendix A, launcher).

The registry adds no authority. A role's authority comes from its cited records, and the registry only collects them. Today's authority by kind is in the next section.

### Minimum viable product

This is the target design. Nothing in it is built now except what's already in flight: the Handler field with Operator and Agent, the `area:` label on agent Requests, and the capability vocabulary used as suggestions only. The registry, capability bundles, composition at launch, and the plan below set up later features.

## How it works

### Role kinds and today's authority

Each row is today's authority as the records state it ([appendix A](A-evidence.md) has every source, labeled existing). The operator row is for reference and isn't a registry kind. The builder row carries the authority today's per-repository owners hold, held per Request in the target design. Two kinds of record grant authority. One is the operator's own decision. The other is a reversible call by the coordinator, made within authority the operator gave.

| Kind | Scope | Decides | May merge or approve | May never | Requests reach it by |
| --- | --- | --- | --- | --- | --- |
| Operator | Everything | The reserved matters: the Laws, irreversible destruction of what tbhb-dev didn't create, destroying the last working copy, and adding or widening authority over decisions | Anything. Can stop any agent | Not limited by the registry | Operator Requests (Handler: Operator). The operator's list of open items is the live path until cutover |
| Coordinator | All tbhb-dev repositories, for priorities and routing | Overall priorities and capacity, questions that cross roles, its own workings, and 1Password Connect grants | Approves Normal Changes that cross roles. Pull request approvals go through the approver, the narrower of two readings in appendix A. Merges nothing it doesn't own | Approve its own change, act as an approval step for owners, weaken a check that reviews it, or widen anyone's authority | Session messages today. Later, agent Requests whose `cap:` labels suggest its capabilities |
| Approver | All tbhb-dev repositories | Whether a green head meets the approval checks | Approves with the approval command at one exact head, after reading the full diff, seeing `conclusion` succeed, and seeing two reviewer families clean at that head | Approve a pull request it or the coordinator opened, or approve where a Law, authority, or a credential is touched, the operator holds the item, the maker reviewed itself, or reviewers disagree. Those go to the coordinator | A one-line green notice from the builder, naming the repository, number, full head, and merger |
| Reviewer, today as review launchers and review runs | The repositories on each launcher's list | Which reviewer pair runs, per the model roster's review pairs, launched for every green head at once | Neither. Reviews post as `COMMENT` | Launch a reviewer-role run for a repository it owns, approve, or merge | Review requests from builders today. Later, `pull_request` events |
| Builder, today held by per-repository owner sessions | Today, the owner session's repositories. In the target, the repository a Request's `area:` label names | Its own work: what's next, hosts, retries, fixes, reviewer findings, deploy steps, and two-way doors in its area | Merges its own pull requests as its repository's ruleset allows: after the approval lands where one is required, matching the head, and on green where none is, as in the wiki repositories. Deploys its own merged services until the deploy runner works | Approve its own change, or pick its own reviewer | Messages today. In the target, agent Requests with no matching `cap:` suggestion fall back to it |
| Admin | Organization settings, consoles, and credential issuing | How to carry out admin steps others decided | Neither. The admin App never approves | Weaken a ruleset or add a bypass actor, change membership or roles, delete, transfer, or archive a repository, turn on paid features, or make a private repository public | Messages today. In the target, agent Requests whose `cap:` labels suggest its capabilities |

[Appendix C](C-registry.md#todays-kinds) lists each kind's capabilities and cites each line. Host work runs today through the ops account and its two playbook capabilities, which belong to the account layer, not to a kind.

### Generated and checked outputs

| Output | Today | From the registry | Who applies it |
| --- | --- | --- | --- |
| Request labels on agent Requests | Not yet created. The operator decided the Handler field keeps two values, Operator and Agent, and an agent Request carries an `area:<repo>` label and `cap:<name>` labels, which suggest routing and never block it | Generated: one `area:<repo>` label per tbhb-dev repository and one `cap:<name>` label per capability in the vocabulary. A check compares the labels with the registry, refuses only a `cap:` label outside the vocabulary, and confirms the Handler field holds Operator and Agent | The label sync creates the labels, in each repository that takes Requests or organization-wide. The records role owns the Request template |
| CODEOWNERS | One owner-less `*` line in every repository, by the operator's decision. All agents act as one GitHub user, so a line can't name a role | Checked as is. The records repository also lists the authors allowed for decision auto-merge, which the operator put in CODEOWNERS | The rules baseline sync |
| Review-launcher check | A built-in list in the launcher's code plus a state-directory file. Both map session names to repositories, and that mapping is interim | Checked against the reviewer kind: a reviewer-kind run starts only from a workflow, never from the builder run of the same Request. A check fails when the built-in list disagrees | Tools for the code, and the deployer for the file |
| Standup participants | A hand-kept file of participants and their repositories on the coordinator's host | Generated from the kinds with `standup = true`. Interim sessions stay in the file until they retire | The coordinator |
| Launch definitions | None. Sessions start by hand, and runs get a brief | The base generated per kind and harness, and composed with the Request's capabilities per run ([appendix D](D-harnesses.md)) | The deployer, at user level on every agent host |

### From a role to a launch

A kind's launch defaults are a harness, a model, an effort, a launcher profile (`build` or `review`), and its base bundle. In the target, a capability is a bundle of run configuration (allow and deny rules, hooks, task-scoped credentials, and context fragments), and `agent exec --role <kind>` composes the base with the Request's capabilities that the kind may take into that run's definition (R31).

| Harness | Definition | How a launch selects it | Guardrails it can carry |
| --- | --- | --- | --- |
| Claude Code | An agent definition with `model`, `effort`, `tools`, `disallowedTools`, `hooks`, and others | `--agents <file>` with `--agent <role>`, so the main thread takes the definition's tools and model | Tool allow and deny lists, and hooks. Account denies still apply, and a definition can't lift them |
| Codex | A profile file, `$CODEX_HOME/<role>.config.toml`, with `developer_instructions`, model, effort, and feature switches, plus a rules file | `-p <role>` in the run's private `CODEX_HOME` | Prefix rules in `$CODEX_HOME/rules/`. Custom agent files exist, but only for spawned subagents |
| Antigravity | A custom agent Markdown file with `tools`, `model`, `commandExecutionPolicy`, and `hooks` | `--agent <role>`, which works in headless runs | Tool lists and hooks. How these combine with settings denies is unverified |

Three facts limit Claude Code guardrails under the launcher as it runs today. Full-access runs set `disableAllHooks`, run in `bypassPermissions`, and load only user settings. A definition's `permissionMode` is ignored under a bypass parent, while deny rules block in every mode. Whether frontmatter hooks survive `disableAllHooks`, and whether `--agent` applies a definition's hooks to the main thread, are unverified. Step 1 of the plan probes both. [Appendix D](D-harnesses.md) has the detail and the field-by-field rules the generator follows: it emits no field that widens authority, so it leaves out `permissionMode`, `mcpServers`, and any hook outside the guard list.

### From sessions to roles

Today the coordinator, the approver, the review launchers, and per-repository owners run as long-running interactive sessions, named by their terminal pane titles. The operator decided that the per-repository sessions are interim and retire as loops replace them. The operator also decided that a deterministic loop should replace the long-lived coordination session, that the end state has no coordination sessions, and that asks between roles move onto Requests. Tools already has the loop's runner in shadow mode, with an approval command setting, and it runs nothing live yet. The registry is the bridge, in three steps:

1. **Interim.** A long-running session starts from its kind's definition (`claude --agent <kind>`), so the session and a launch share one source.
2. **Launch per task.** Interim sessions and workflows start runs with `--role <kind>`. The launcher fills the defaults, and the review-launcher check keys on the role, not on a pane title.
3. **Launch per Request.** The event gateway delivers a Request event to a workflow, and the workflow launches a run of the kind whose capabilities the `cap:` labels suggest, with the context of the repository the `area:<repo>` label names. The labels never block routing: with no match, the workflow launches the builder kind. A kind needs no session.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| The registry is read as granting authority | A role gains authority through an edit nobody recognized as a grant | Every authority line cites a record, and the check refuses a widened line without an operator record | New |
| Claude Code frontmatter hooks don't run under the launcher's `disableAllHooks`, or `--agent` drops them | A role's hook guardrails don't bind launched runs | Step 1 probes it. Command limits stay in account settings denies, which bind in every mode | Unverified |
| A definition field widens: `permissionMode`, `mcpServers`, or a hook that approves | A role gets more than the account allows | The generator emits no field that widens authority, and the check rejects those fields | Documented behavior, appendix D |
| Codex `-c` overrides from the launcher beat a profile file's values | A role's model or instructions are silently replaced | The launcher passes role values once, through the profile, and step 1 tests the precedence | Unverified |
| The review-launcher list keys on pane titles, which a session can forge | A wrong session launches reviews | It's documented as a guard against mistakes. Keying on the role definition doesn't change that, and the approver's checks stay the barrier | Existing |
| Antigravity custom agents may loosen settings denies | A role escapes a deny in Antigravity runs | Antigravity runs stay review and plan-mode work until a probe settles it | Unverified |
| A Request's `cap:` suggestions are wrong | A run of the wrong kind starts | The run or the router relabels or reassigns it, and with no match the builder kind takes it | New |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | Probe: Claude Code `--agents` and `--agent` under the launcher's flags, with hooks on and off, in a `-p` run; Codex `-p` profile precedence against `-c`; Antigravity `--agent` with `--dangerously-skip-permissions` | A probe report with log paths marks each unverified row in appendix D proven or refuted | Tools, with the harness configuration owner |
| 2 | `roles/registry.toml`, its schema, and the registry check, with today's kinds only | The check passes, and two reviews confirm each authority line against its record | Harness configuration owner |
| 3 | Request label generator and live check | The check reads the `area:` and `cap:` labels and the Handler field, and both match the registry | Harness configuration owner, then the label sync applies |
| 4 | Review-launcher check against the reviewer kind | The check fails on a planted mismatch between the built-in list and the reviewer kind | Tools |
| 5 | Standup participant kinds generated | The standup opens from the generated file plus the interim sessions | Coordinator |
| 6 | Harness definitions rendered and deployed, at user level for Claude Code and Antigravity and at the root-owned copy path for Codex | The deployer's check matches each installed file's hash with the rendered file on every agent host, and one probe run per harness reports the role's model | Harness configuration owner, with ops for the deploy |
| 7 | `agent exec --role <kind>` reads the registry defaults and passes the definition | A run started with only `--role` and a brief records the kind's harness, model, and effort | Tools |
| 8 | Interim sessions start from their kind's definition | Each session's start command names `--agent <kind>` | Coordinator |

Recommended first piece: step 1. It is small and read-only, and its answers decide how much of the guardrail design binds launched runs.

## Appendices

- [Decisions](decisions.md): 1 open, then the decided record.
- [A. Evidence](A-evidence.md): every source, labeled existing, new, or unverified.
- [B. Alternatives considered](B-alternatives.md): the registry's home and format, and per-harness options not taken.
- [C. Registry](C-registry.md): schema, today's entries, and the checks.
- [D. Harness definitions](D-harnesses.md): Claude Code, Codex, and Antigravity fields, how each combines with account settings, and the launcher's flags.
