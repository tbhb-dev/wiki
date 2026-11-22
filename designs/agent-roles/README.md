---
type: Design Draft
title: Agent roles
description: One registry of tbhb-dev's agent roles, their authority today, and how each role becomes a launch in Claude Code, Codex, and Antigravity.
tags: [agent-roles, authority, launcher, harness, design]
status: draft
owner: Harness configuration owner
updated: "2026-11-22"
decisions_open: 2
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore unsandboxed sessionless cutover -->

# Agent roles

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Harness configuration owner | 2026-11-22 | 2 open decisions in [decisions.md](decisions.md), about 5 minutes |

## Summary

tbhb-dev's roles live in many places today: decision records, the launcher's code, a review-launcher file, a standup participants file, rulesets, and the habits of long-running sessions. Each copy drifts on its own, and none of them is a definition a workflow can launch.

This design writes today's roles into one machine-readable registry and widens no one's authority. Every role-shaped list is generated from the registry or checked against it: the `role:<name>` labels on agent Requests, CODEOWNERS, the review-launcher scopes, the standup participants, and one launch definition per role for each harness.

For the operator, the change is that a role becomes a file that can be read and reviewed. A role can then be launched for each task or Request, which is the sessionless direction the operator set.

## Proposal

When this is built, these pieces exist:

- **The registry**, `roles/registry.toml` in the private harness configuration repository, owned by the harness configuration owner. Each entry names a role, its kind, its scope (repositories, domains, and host classes), what it decides, what it may merge or approve, what it may never do, how Requests reach it, and its launch defaults. Each authority line cites the record that grants it. [Appendix C](C-registry.md) gives the schema and today's entries.
- **A registry check**, a Bazel test in that repository. It fails when an authority line has no citation, when a kind's limits are broken, or when an entry gains a decide, approve, or merge value that main lacks and the change carries no operator record.
- **Generators**, commands in tools. They write the generated files and check the ones that stay hand-kept, one output each ([How it works](#generated-and-checked-outputs)).
- **Harness definitions**, rendered per role: a Claude Code agent definition, a Codex profile file with its rules file, and an Antigravity custom agent. The harness configuration deployer installs the Claude Code and Antigravity definitions at user level on every agent host, and writes the Codex files to a root-owned path that the launcher copies into each run's private `CODEX_HOME`. [Appendix D](D-harnesses.md) maps the fields.
- **Launcher support** for `tbhbdev alpha agent exec --role <name>`. The launcher reads the role's defaults and passes the role's definition to the harness. The flag already exists and is recorded, but only `reviewer` changes behavior today (appendix A, launcher).

The registry adds no authority. A role's authority comes from its cited records, and the registry only collects them. Today's authority by kind is in the next section.

## How it works

### Role kinds and today's authority

Each row is today's authority as the records state it ([appendix A](A-evidence.md) has every source, labeled existing). Two kinds of record grant authority. One is the operator's own decision. The other is a reversible call by the coordinator, made within authority the operator gave.

| Kind | Scope | Decides | May merge or approve | May never | Requests reach it by |
| --- | --- | --- | --- | --- | --- |
| Operator | Everything | The reserved matters: the Laws, irreversible destruction of what tbhb-dev didn't create, destroying the last working copy, and adding or widening authority over decisions | Anything. Can stop any agent | Not limited by the registry | Operator Requests (Handler: Operator). The operator's list of open items is the live path until cutover |
| Coordinator | All tbhb-dev repositories, for priorities and routing | Overall priorities and capacity, questions that cross roles, its own workings, and 1Password Connect grants | Approves Normal Changes that cross roles. Pull request approvals go through the approver, the narrower of two readings in appendix A. Merges nothing it doesn't own | Approve its own change, act as an approval step for owners, weaken a check that reviews it, or widen anyone's authority | Session messages today. Agent Requests with its `role:` label, per the Request decisions |
| Approver | All tbhb-dev repositories | Whether a green head meets the approval checks | Approves with the approval command at one exact head, after reading the full diff, seeing `conclusion` succeed, and seeing two reviewer families clean at that head | Approve a pull request it or the coordinator opened, or approve where a Law, authority, or a credential is touched, the operator holds the item, the maker reviewed itself, or reviewers disagree. Those go to the coordinator | A one-line green notice from the owner, naming the repository, number, full head, and merger |
| Review launcher | The repositories on its launcher list | Which reviewer pair runs, per the model roster's review pairs, launched for every green head at once | Neither. Reviews post as `COMMENT` | Launch a reviewer-role run for a repository it owns, approve, or merge | Review requests from owners. Later, `pull_request` events |
| Owner, per repository or domain | Its own repositories, or its domain such as hosts and deploys | Its own work: what's next, hosts, retries, fixes, reviewer findings, deploy steps, and two-way doors in its area | Merges its own pull requests as its repository's ruleset allows: after the approval lands where one is required, matching the head, and on green where none is, as in the wiki repositories. Deploys its own merged services until the deploy runner works | Approve its own change, pick its own reviewer, or decide for another owner's area | Agent Requests with its `role:` label, and messages today |
| Admin | Organization settings, consoles, and credential issuing | How to carry out admin steps others decided | Neither. The admin App never approves | Weaken a ruleset or add a bypass actor, change membership or roles, delete, transfer, or archive a repository, turn on paid features, or make a private repository public | Requests to the admin role, and messages today |

[Appendix C](C-registry.md#todays-entries) lists the entries this produces, such as the operations owner, whose runs may use the ops account and that account's two playbook capabilities. It cites each line.

### Generated and checked outputs

| Output | Today | From the registry | Who applies it |
| --- | --- | --- | --- |
| Role labels on agent Requests | Not yet created. The operator decided the Handler field keeps two values, Operator and Agent, and an agent Request names its role with a `role:<name>` label | Generated: one `role:<name>` label per role with `role_label = true`. A check compares the labels in each repository with the registry, and the Handler field with Operator and Agent | The label sync creates the labels, in each repository that takes Requests or organization-wide. The records role owns the Request template |
| CODEOWNERS | One owner-less `*` line in every repository, by the operator's decision. All agents act as one GitHub user, so a line can't name a role | Checked as is. The records repository also lists the authors allowed for decision auto-merge, which the operator put in CODEOWNERS | Each repository's owner, through the rules baseline sync |
| Review-launcher scopes | A built-in list in the launcher's code plus a state-directory file. Both map session names to repositories | Generated: each review launcher's repositories, and each repository's owner role. A check fails when the built-in list disagrees | Tools for the code, and the deployer for the file |
| Standup participants | A hand-kept file of participants and their repositories on the coordinator's host | Generated from the roles with `standup = true` | The coordinator |
| Launch definitions | None. Sessions start by hand, and runs get a brief | Generated per role and harness ([appendix D](D-harnesses.md)) | The deployer, at user level on every agent host |

### From a role to a launch

A role's launch defaults are a harness, a model, an effort, a launcher profile (`build` or `review`), and its guardrails. `tbhbdev alpha agent exec --role <name>` fills any of these the caller leaves out, records the role, and passes the definition.

| Harness | Definition | How a launch selects it | Guardrails it can carry |
| --- | --- | --- | --- |
| Claude Code | An agent definition with `model`, `effort`, `tools`, `disallowedTools`, `hooks`, and others | `--agents <file>` with `--agent <role>`, so the main thread takes the definition's tools and model | Tool allow and deny lists, and hooks. Account denies still apply, and a definition can't lift them |
| Codex | A profile file, `$CODEX_HOME/<role>.config.toml`, with `developer_instructions`, model, effort, and feature switches, plus a rules file | `-p <role>` in the run's private `CODEX_HOME` | Prefix rules in `$CODEX_HOME/rules/`. Custom agent files exist, but only for spawned subagents |
| Antigravity | A custom agent Markdown file with `tools`, `model`, `commandExecutionPolicy`, and `hooks` | `--agent <role>`, which works in headless runs | Tool lists and hooks. How these combine with settings denies is unverified |

Three facts limit Claude Code guardrails under the launcher as it runs today. Full-access runs set `disableAllHooks`, run in `bypassPermissions`, and load only user settings. A definition's `permissionMode` is ignored under a bypass parent, while deny rules block in every mode. Whether frontmatter hooks survive `disableAllHooks`, and whether `--agent` applies a definition's hooks to the main thread, are unverified. Step 1 of the plan probes both. [Appendix D](D-harnesses.md) has the detail and the field-by-field rules the generator follows: it emits no field that widens authority, so it leaves out `permissionMode`, `mcpServers`, and any hook outside the guard list.

### From sessions to roles

Today the coordinator, the approver, the review launchers, and most owners run as long-running interactive sessions, named by their terminal pane titles. The operator decided that a deterministic loop should replace the long-lived coordination session, that the end state has no coordination sessions, and that asks between roles move onto Requests. Tools already has the loop's runner in shadow mode, with an approval command setting, and it runs nothing live yet. The registry is the bridge, in three steps:

1. **Interim.** A long-running session starts from its role's definition (`claude --agent <role>`), so the session and a launch share one source.
2. **Launch per task.** Owners and workflows start runs with `--role <name>`. The launcher fills the defaults, and the review-launcher check keys on the role, not on a pane title.
3. **Launch per Request.** The event gateway delivers a Request event to a workflow, and the workflow launches the run whose role matches the `role:<name>` label on an agent Request. A role that has no events left to handle needs no session.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| The registry is read as granting authority | A role gains authority through an edit nobody recognized as a grant | Every authority line cites a record, and the check refuses a widened line without an operator record | New |
| Claude Code frontmatter hooks don't run under the launcher's `disableAllHooks`, or `--agent` drops them | A role's hook guardrails don't bind launched runs | Step 1 probes it. Command limits stay in account settings denies, which bind in every mode | Unverified |
| A definition field widens: `permissionMode`, `mcpServers`, or a hook that approves | A role gets more than the account allows | The generator emits no field that widens authority, and the check rejects those fields | Documented behavior, appendix D |
| Codex `-c` overrides from the launcher beat a profile file's values | A role's model or instructions are silently replaced | The launcher passes role values once, through the profile, and step 1 tests the precedence | Unverified |
| The review-launcher list keys on pane titles, which a session can forge | A wrong session launches reviews | It's documented as a guard against mistakes. Keying on the role definition doesn't change that, and the approver's checks stay the barrier | Existing |
| Antigravity custom agents may loosen settings denies | A role escapes a deny in Antigravity runs | Antigravity runs stay review and plan-mode work until a probe settles it | Unverified |
| The registry holds private repository names and host classes | A leak into a public repository | The registry lives in a private repository, and public repositories get only their own generated files | New |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | Probe: Claude Code `--agents` and `--agent` under the launcher's flags, with hooks on and off, in a `-p` run; Codex `-p` profile precedence against `-c`; Antigravity `--agent` with `--dangerously-skip-permissions` | A probe report with log paths marks each unverified row in appendix D proven or refuted | Tools, with the harness configuration owner |
| 2 | `roles/registry.toml`, its schema, and the registry check, with today's entries only | The check passes, and two reviews confirm each authority line against its record | Harness configuration owner |
| 3 | Role label generator and live check | The check reads each repository's labels and the Handler field, and both match the registry | Harness configuration owner, then the label sync applies |
| 4 | Review-launcher scopes generator and the built-in list check | The launcher file is generated, and the check fails on a planted mismatch | Tools |
| 5 | Standup participants generated | The standup opens from the generated file | Coordinator |
| 6 | Harness definitions rendered and deployed, at user level for Claude Code and Antigravity and at the root-owned copy path for Codex | The deployer's check matches each installed file's hash with the rendered file on every agent host, and one probe run per harness reports the role's model | Harness configuration owner, with ops for the deploy |
| 7 | `agent exec --role` reads the registry defaults and passes the definition | A run started with only `--role` and a brief records the role's harness, model, and effort | Tools |
| 8 | Long-running sessions start from their role definitions | Each session's start command names `--agent <role>` | Coordinator |

Recommended first piece: step 1. It is small and read-only, and its answers decide how much of the guardrail design binds launched runs.

## Appendices

- [Decisions](decisions.md): 2 open, then the decided record.
- [A. Evidence](A-evidence.md): every source, labeled existing, new, or unverified.
- [B. Alternatives considered](B-alternatives.md): the registry's home and format, and per-harness options not taken.
- [C. Registry](C-registry.md): schema, today's entries, and the checks.
- [D. Harness definitions](D-harnesses.md): Claude Code, Codex, and Antigravity fields, how each combines with account settings, and the launcher's flags.
