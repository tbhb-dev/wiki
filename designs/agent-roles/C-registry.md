---
type: Design Appendix
title: Agent roles registry
description: The registry's role kinds and capabilities, its schema, routing, and the checks that keep it from widening authority.
tags: [agent-roles, registry, design]
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

# Appendix C. Registry

Back to [the design](README.md).

Everything in this appendix is new unless it says existing. The registry is planned and doesn't exist yet.

## Kinds and capabilities only

The registry defines role kinds and capabilities, and nothing per repository, by the operator's decisions on D1 (R27 to R30). No role is assigned to a repository. A capability is a bundle of run configuration, and a kind is a base plus the capabilities it may take, by the operator's direction (R31). Creating a kind or a capability, widening one, or letting a kind take another capability is the operator's, so every kind and capability line cites an operator record.

The operator is not a kind. The operator holds every authority, and the registry doesn't limit it.

## Schema

One file, `roles/registry.toml`, in the private harness configuration repository, beside the mechanical parts that go into harnesses: generated agent definitions and profiles, allow and deny entries, and hooks (D8). In the target design, the public kinds-and-capabilities page in this wiki is generated from it, carrying names, kinds, the capability vocabulary, and descriptions only, never allow or deny entries, hooks, credential scopes, or paths, and a check fails when the page differs from the registry.

```toml
schema_version = 1

[vocabulary]
# The fixed capability list. Each becomes a cap:<name> label and has one [capability.<name>] bundle.
capabilities = ["repo-write", "merge", "approve", "change-approve", "review", "credential-grant", "org-settings", "console", "host-root"]

[capability.merge]
summary = "Merge a pull request the Request produced, at its approved head."
allow = ["<allow entry>"]        # emitted only where it narrows the base; never lifts a deny
deny = []                       # composed in; adds to the account's denies
hooks = ["<hook>"]              # names from the fixed guard list
credentials = ["<credential scope>"]   # task-scoped: the Request's area repository only
context = ["<context fragment>"]   # fragments appended to the definition's instructions
authority = ["R9", "R31"]

[kind.builder]
summary = "Does the work a Request asks for, in the area the Request names."
decides = ["own-work", "two-way-doors-in-scope", "deploy-own-services"]
merges = "own-per-ruleset"      # none | own-per-ruleset
approves = "none"               # none | approval-rules | normal-changes-across-roles
never = ["approve-own-change", "pick-own-reviewer"]
may_take = ["repo-write", "merge"]   # capabilities this kind may take; the operator grants these
fallback = true                 # routing falls back to this kind
standup = false                 # whether runs of this kind report at standup
authority = ["R9", "R10", "R11", "R26", "R30"]   # operator records, or coordinator records that rest on one

[kind.builder.launch]
harness = "codex"
model = "gpt-6.2-sol"
effort = "high"
profile = "build"               # the launcher profile: build | review

[kind.builder.base]
# The base bundle every run of the kind gets, in the same shape as a capability.
allow = []
deny = []
hooks = []
credentials = ["<credential scope>"]
context = ["<context fragment>"]

[record.R10]
summary = "Owners decide their own work."
who = "operator"                # operator | coordinator; a coordinator record also sets rests_on = ["<operator record>"]
date = "2026-11-15"
source = "operator decision notes, owners decide their own work"
```

The value lists are closed. A new value for `decides`, `merges`, `approves`, or `never` is a schema change, reviewed as one. No table names a repository, so nothing in the registry grows with the repository count. A credential scope resolves at launch to the repository the Request's `area:` label names.

## Routing and context

An agent Request carries the Handler value Agent, one `area:<repo>` label, and `cap:<name>` labels for what fulfilling it takes.

- **`area:<repo>`** scopes which repository's context a run loads: its clone, its rules, and its records. It doesn't pick who handles the Request (R30).
- **`cap:` labels** are suggestions, not constraints (R29). Routing matches them to a kind that may take them, and a workflow launches a run of that kind with the area's context. The run or the router may relabel or reassign the Request. Routing never blocks on the labels: with no match, it launches the fallback kind, builder.
- **Composition at launch:** the launcher composes the kind's base with the Request's capabilities that the kind may take, and renders the result into the harness's definition for that run ([appendix D](D-harnesses.md#capability-bundles-per-harness)). A suggested capability the kind may not take is dropped and noted in the run record, never an error. Injecting a capability the kind may take is routine and needs no record beyond the run's.
- **No kind may take `host-root`** in the first registry. Host work runs today through the ops account, which belongs to the account layer, not to a kind.

## Today's kinds

Each line cites the key from [appendix A](A-evidence.md#operator-and-coordinator-records). The builder kind carries the authority that per-repository owners hold today, held per Request instead of per repository. Its name is a proposal for the operator to confirm at merge.

| Kind | Decides | Merges | Approves | May take | Authority |
| --- | --- | --- | --- | --- | --- |
| builder | Its own work on the Request, two-way doors in the area, its deploys until the deploy runner works | A pull request its Request produced, as the repository's ruleset allows | None | `repo-write`, `merge` | R9, R10, R11, R26, R30 |
| approver | Whether a green head meets the checks | None | `approval-rules` | `approve` | R1, R2, R9 |
| reviewer | Which reviewer pair runs, when launching reviews. The verdict, as a review run | None | None. Reviews post as `COMMENT` | `review` | R22, R23 (a coordinator record resting on the operator's direction recorded in it) |
| coordinator | Priorities, cross-role questions, its own workings, Connect grants | None outside its own work | `normal-changes-across-roles` | `change-approve`, `credential-grant` | R0, R2, R7, R8, R10, R17 |
| admin | How to carry out admin steps others decided | None | None | `org-settings`, `credential-grant`, `console` | R5, R6, R7 |

## Today's sessions are interim

Today, per-repository and per-domain owner sessions run beside the coordinator, the approving session, and the review sessions (R10, R11). The operator decided they are interim and retire as loops replace them (R30). The registry doesn't record them. While they run, they act with the builder kind's authority for their repositories, and the interim mapping of sessions to repositories stays in the launcher's state file and the standup participants file.

### What every kind may never do

These come from the constitution (R0), so every kind inherits them and the registry doesn't repeat them:

- Breach a Law, or widen anyone's authority over decisions, its own included.
- Approve its own change, pick its own reviewer, or let a reviewer share the maker's context.
- Read, change, or delete the operator's own data beyond the access given.
- Resist or work around a stop.

### Admin limits

The admin kind's `never` list is the refusal list of the tool that mints the admin App's tokens (R5): weakening a ruleset or branch protection, or adding a bypass actor; changing organization membership, collaborators, teams, or roles; deleting, transferring, or archiving a repository; turning on paid features; and changing a private repository to public.

## Checks

The registry check is a Bazel test in the private harness configuration repository. It fails when:

- A kind's `authority` is empty, names a key with no `[record]` table, or names a record whose decider is neither the operator nor the coordinator with a `rests_on` that names an operator record.
- A change adds a kind or a capability, widens a capability's bundle, or gives a kind a `decides`, `merges`, `approves`, or `may_take` value that main lacks, and no `[record]` with `who = "operator"` was added for it.
- A bundle widens: it carries a field the harness mapping never emits, a hook outside the guard list, a credential scope wider than the Request's area repository, an allow entry wider than the base's, or an allow rule that matches an account deny.
- A kind breaks its limits: a reviewer can't have `merges` or `approves`, an approver can't have `merges`, and exactly one kind has `fallback = true`.
- Any table names a repository or a host.
- A bundle field names a value that isn't on the allowed list in [appendix D](D-harnesses.md#what-the-generator-may-emit).

The generators each carry their own check, run in CI where the output lives, and CODEOWNERS, which isn't generated, is checked as is:

- **Request labels:** compare the label set with one `area:<repo>` label per tbhb-dev repository, read from the organization, plus one `cap:<name>` label per vocabulary entry. On Requests, it checks only that each `cap:` label is in the vocabulary, and never refuses a Request for missing, extra, or mismatched ones. The same check confirms the Handler field still holds exactly Operator and Agent.
- **CODEOWNERS:** compare each repository's file with the baseline's owner-less `*`, plus the records repository's allowed-author lines. Kinds can't be code owners, because all agents act as one GitHub user.
- **Review-launcher check:** compare the launcher's built-in list with the reviewer kind. In the target, a reviewer-kind run starts only from a workflow, never from the builder run of the same Request. The interim list of review sessions stays until those sessions retire.
- **Standup participants:** generate the participant kinds from the kinds where `standup` is true. Interim sessions stay in the participants file until they retire.
- **Harness definitions:** render each kind and compare each installed file's hash with the rendered one on each agent host.

## How a change flows

1. The harness configuration owner opens a pull request on the registry, carrying the operator's record for any kind or capability change.
2. The registry check runs. A new or widened kind or capability without an operator record fails.
3. Two reviewer families review it, under that repository's review policy.
4. On merge, the generators write their outputs. The label sync creates the `area:` and `cap:` labels, the deployer installs harness definitions, and tools releases a launcher whose built-in list matches.
