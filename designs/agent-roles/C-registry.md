---
type: Design Appendix
title: Agent roles registry
description: The registry's schema, today's entries by kind, and the checks that keep it from widening authority.
tags: [agent-roles, registry, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Appendix C. Registry

Back to [the design](README.md).

Everything in this appendix is new unless it says existing. The registry is planned and doesn't exist yet.

## Schema

One file, `roles/registry.toml`, in the private harness configuration repository. Each role is one table.

```toml
schema_version = 1

[role.tools-owner]
kind = "owner"                  # operator | coordinator | approver | review-launcher | owner | admin
summary = "Owns the tools repository and the launcher."
repos = ["tbhb-dev/tools"]      # repositories in scope; "*" means every tbhb-dev repository
domains = []                    # named domains, such as "hosts" or "deploys"
host_classes = []               # host classes the role acts on; never host names
decides = ["own-work", "two-way-doors-in-scope", "deploy-own-services"]
merges = "own-per-ruleset"      # none | own-per-ruleset
approves = "none"               # none | approval-rules | normal-changes-across-roles
never = ["approve-own-change", "pick-own-reviewer", "decide-other-owners-area"]
requests = { handler = true, events = ["issues", "issue_comment"] }
standup = true
authority = ["R9", "R10", "R11", "R26"]   # each value must resolve to an entry under [record]

[role.tools-owner.launch]
harness = "codex"
model = "gpt-6.1-sol"
effort = "high"
profile = "build"               # the launcher profile: build | review
brief = "roles/briefs/tools-owner.md"

[role.tools-owner.guardrails]
deny_commands = []              # rendered into each harness's command denies, adding to the account's
tools = []                      # an allow list; empty means the harness default
disallowed_tools = []
hooks = ["guard-merge-match-head"]   # names from a fixed list of guard hooks in tools

[record.R10]
summary = "Owners decide their own work."
who = "operator"                # operator | coordinator
date = "2026-11-15"
source = "operator decision notes, owners decide their own work"
```

The value lists are closed. A new value for `decides`, `merges`, `approves`, or `never` is a schema change, reviewed as one.

## Today's entries

These are the entries the first registry pull request writes. Each line cites the key from [appendix A](A-evidence.md#operator-and-coordinator-records). Where a role's scope names a private repository, the registry names it, and this public appendix describes it by role.

| Entry | Kind | Scope | Decides | Merges | Approves | Requests | Authority |
| --- | --- | --- | --- | --- | --- | --- | --- |
| operator | operator | `*` | Reserved matters | Any | Any | Handler: Operator | R0 |
| coordinator | coordinator | `*`, for priorities | Priorities, cross-role questions, its own workings, Connect grants | None outside its own work | `normal-changes-across-roles` | Handler option | R0, R2, R7, R8, R10, R17 |
| approver | approver | `*` | Whether a green head meets the checks | None | `approval-rules` | Green notice; later a Handler option | R1, R2, R9 |
| review-launcher | review-launcher | Repositories on its list, never its own | Which reviewer pair runs | None | None | Review requests, later `pull_request` events | R22, R23, tools `launcher.py` |
| `<repo>`-owner, one per repository | owner | Its repositories | Own work, two-way doors in scope, its deploys until the deploy runner works | Own, as the repository's ruleset allows | None | Handler option | R9, R10, R11, R26 |
| ops-owner | owner | Hosts, runners, the deploy runner, inventory, and the private operations repository | Same as an owner. Its runs may use the ops account, whose two playbook capabilities belong to the account, not the role | Own, as the repository's ruleset allows | None | Handler option | R10, and a decision record of the harness configuration repository |
| harness-configuration-owner | owner | The private harness configuration repository, the registry, and account harness settings | Same as an owner | Own, as the repository's ruleset allows | None | Handler option | R10, R21 |
| records-owner | owner | The private IT service management records repository and the Handler option list | Same as an owner | Own, as the repository's ruleset allows. The records App merges decision records on CI alone | None | Handler option | R16, R18, R20 |
| admin | admin | Organization settings, consoles, credential issuing | How to carry out admin steps others decided | None | None | Handler option | R5, R6, R7, R20 |

The standard owner entry is the `<repo>`-owner row with nothing added. D1 in [decisions.md](decisions.md) asks whether the coordinator may add one for each new repository without a Request.

### What every role may never do

These come from the constitution (R0), so every entry inherits them and the registry doesn't repeat them:

- Breach a Law, or widen anyone's authority over decisions, its own included.
- Approve its own change, pick its own reviewer, or let a reviewer share the maker's context.
- Read, change, or delete the operator's own data beyond the access given.
- Resist or work around a stop.

### Admin limits

The admin entry's `never` list is the refusal list of the tool that mints the admin App's tokens (R5): weakening a ruleset or branch protection, or adding a bypass actor; changing organization membership, collaborators, teams, or roles; deleting, transferring, or archiving a repository; turning on paid features; and changing a private repository to public.

## Checks

The registry check is a Bazel test in the private harness configuration repository. It fails when:

- An entry's `authority` is empty, or names a key with no `[record]` table.
- An entry breaks its kind's limits. A review launcher can't have `merges` or `approves`. An approver can't have `merges`. Only the operator can have `"*"` for merges.
- A change gives any entry a `decides`, `merges`, or `approves` value that main lacks, or adds an entry, and no `[record]` with `who = "operator"` was added for it. Under D1, a standard owner entry with a coordinator record passes.
- A `guardrails` field names a value that isn't on the allowed list in [appendix D](D-harnesses.md#what-the-generator-may-emit).
- A role names a host instead of a host class.

The generators each carry their own check, run in CI where the output lives:

- **Handler options:** read the live organization issue field with a read-only call and compare it with Operator plus the roles where `requests.handler` is true.
- **CODEOWNERS:** compare each repository's file with the baseline's owner-less `*`, plus the records repository's allowed-author lines.
- **Review-launcher scopes:** compare the launcher's built-in list with the review-launcher entries, and the state-directory file with the generated one.
- **Standup participants:** compare the participants file with the roles where `standup` is true.
- **Harness definitions:** render each role and compare with what the deployer installed on each agent host.

## How a change flows

1. A role's owner, or the coordinator for a new owner role, opens a pull request on the registry with its record.
2. The registry check runs. A widened line without an operator record fails.
3. Two reviewer families review it, under that repository's review policy.
4. On merge, the generators write their outputs. The admin role applies Handler options, the deployer installs harness definitions, and tools releases a launcher whose built-in list matches.
