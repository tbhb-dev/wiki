---
type: Design Appendix
title: Status checks for issues evidence
description: What the repositories, schemas, command-line code, and GitHub documentation show for the issue checks design.
tags: [issues, checks, evidence, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Status checks for issues evidence

Back to [the design](README.md).

Each claim carries one label. **Proven** means a test run or merged pull request demonstrated it. **Existing** means current code in a repository or official GitHub documentation states it, read on 2026-11-25. **New** means this design proposes it. **Unverified** means no run has tested it yet, and the plan names the step that checks it.

## GitHub platform facts

| Claim | Label | Source |
| --- | --- | --- |
| The `projects_v2_item` webhook event fires on organization webhooks and GitHub Apps for `created`, `edited`, `converted`, `reordered`, `archived`, `restored`, and `deleted` actions, and requires at least read access to the `Projects` organization permission | existing | [GitHub Docs, Webhook events and payloads: `projects_v2_item`](https://docs.github.com/en/webhooks/webhook-events-and-payloads#projects_v2_item) |
| GitHub Actions does not support `projects_v2_item` or `projects_v2` as a workflow trigger in `on:`, so board moves reach automation only through an organization webhook or an API read | existing | [GitHub Docs, Events that trigger workflows](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows) |
| GitHub Projects built-in workflows that set a field value (when an item is added, reopened, closed, or merged, including the two default workflows that set `Done`) only set the built-in `Status` field, plus auto-add and auto-archive actions | existing | [GitHub Docs, built-in project workflows](https://docs.github.com/en/issues/planning-and-tracking-with-projects/automating-your-project/using-the-built-in-automations) |
| Events triggered by a workflow's default `GITHUB_TOKEN` don't start a new workflow run, whereas events from a GitHub App installation token do unless filtered by `sender` | existing | [GitHub Docs, Automatic token authentication](https://docs.github.com/en/actions/concepts/security/github_token) |
| GitHub REST rate limits cap content-creating requests at 80 per minute and 500 per hour | existing | [GitHub Docs, Rate limits for the REST API](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api) |
| Any user or token with `issues: write` or triage permission on a repository can add or remove any label through the REST or GraphQL API, because GitHub offers no per-label write ACL | existing | [GitHub Docs, Manage labels](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/managing-labels) |
| An environment branch policy restricted to `main` denies non-default branch runs before assigning a runner or exposing secrets | proven | [Renovate auto-approval environment probe](../renovate-auto-approval/A-evidence.md#the-environment-probe) |
| Round-trip latency for `tbhb-dev/event-gateway` to receive a `projects_v2_item` webhook and revert a field value via GraphQL | unverified | Measured in plan step 5 |

## Schemas and templates in pm

Read from `tbhb-dev/pm` at commit `3d56b04` on `main` and commit `8603321` (`tbhb-dev/pm#32`, prefixed list IDs):

| Claim | Label | Source |
| --- | --- | --- |
| `tbhb-dev/pm` defines Configure Unify Execute (CUE) issue variant schemas in `schemas/issue.<type>.cue` (`chore`, `defect`, `epic`, `feature`, `imported`, `initiative`, `research`, `spike`, and `task`) plus shared rules in `schemas/issue-sections.cue` | existing | `schemas/` in `tbhb-dev/pm` at `8603321` |
| Starting Markdown templates live in `templates/<type>.md` and render required sections from supplied keys, while constraints live in the CUE schemas | existing | `templates/` and `docs/issue-templates.md` in `tbhb-dev/pm` at `8603321` |
| List items in Acceptance criteria, Verification, and Defect Reproduction use one-based prefixed identifiers (`AC1:`, `V1:`, `S1:`) checked by `#_PrefixedList` in `schemas/issue-sections.cue` | existing | `schemas/issue-sections.cue` and `docs/issue-templates.md` in `tbhb-dev/pm` at `8603321` |
| Each variant loads alongside `core.cue`, `taxonomy.cue`, and `issue.cue` from `tbhb-dev/tools` because every variant defines `#Variant` in package `content` | existing | `docs/issue-templates.md` in `tbhb-dev/pm` and [pm content model](../pm/A-content-model.md#the-files) |
| Seven deliberate mutations of a feature issue each fail `cue vet` on the normalized block model with the expected finding | proven | [pm content model](../pm/A-content-model.md#the-block-model) |
| `Out of scope` is required in every work issue type and records a deliberate decision, where a single `None.` item is valid when nothing is excluded | new | [tbhb-dev/wiki#194](https://github.com/tbhb-dev/wiki/pull/194) at commit `60b35f3`, amending [Definition of Ready](../issue-refinement/B-definition-of-ready.md#readiness-checks) |

## Content kernel and command-line tool in tools

Read from `tbhb-dev/tools` at commit `c23b698` (`tbhb-dev/tools#394`):

| Claim | Label | Source |
| --- | --- | --- |
| `tbhbdev.content_model.normalize` parses Markdown into a JSON block model with section headings, list items, inline spans, and visible word and source character metrics | existing | `src/tbhbdev/content_model.py` in `tbhb-dev/tools` at `c23b698` |
| `tbhbdev.content_vet.check` runs `cue vet --all-errors` against `core.cue`, `issue.cue`, `taxonomy.cue`, and an optional variant schema, then checks parent issue types and title warnings | existing | `src/tbhbdev/content_vet.py` in `tbhb-dev/tools` at `c23b698` |
| `tbhbdev.content` runs the gitleaks secret scan and private-detail checks before writing to GitHub, and exposes `create_issue`, `link_sub_issue`, `link_dependency`, and `add_to_project` | existing | `src/tbhbdev/content.py` in `tbhb-dev/tools` at `c23b698` |
| `tbhbdev alpha issue create` in `src/tbhbdev/alpha/close.py` requires an active agent session trailer, calls `validate_issue()` synchronously, and refuses any body with findings before calling `content.create_issue` | existing | `src/tbhbdev/alpha/close.py` in `tbhb-dev/tools` at `c23b698` |
| `tbhbdev alpha agent exec` launches supervised headless agent runs by host, repository, model, effort, profile, and role | existing | `src/tbhbdev/alpha/agent/driver.py` in `tbhb-dev/tools` at `c23b698` |
| Adding `--draft` to `tbhbdev alpha issue create` and `edit` and blocking builder runs in `tbhbdev alpha agent exec` when `check:passing` is absent | new | Proposed in this design, plan step 3 |

## Import and event gateway designs

| Claim | Label | Source |
| --- | --- | --- |
| The staged tracker import holds 784 drafts, and 247 of them carry front-matter `status: Unverified` or `status: Needs info` across Chores, Defects, Features, Problems, Tasks, Epics, Research, and Spikes | existing | Operator direction of 2026-11-25 and [tracker import](../pm/E-import.md) |
| Tracker import and refinement runs share a write budget of at most 200 writes per hour and 20 per minute | existing | [Tracker import throttling](../pm/E-import.md#throttling) |
| `tbhb-dev/event-gateway` receives the organization GitHub webhook, verifies the Hash-based Message Authentication Code (HMAC) signature with `SHA-256`, stores events in NATS JetStream, and routes matching events to targets | existing | [Event gateway design](../event-gateway/README.md) |
| Under event-gateway decision D17, the gateway process holds no credential that reads or writes organization Project fields, and matches issue events by label instead | existing | [Event gateway decisions](../event-gateway/decisions.md#d17-may-the-gateway-hold-a-credential-that-reads-org-project-fields-so-rules-can-match-a-requests-handler) |
| Incidents and Changes open complete through their verbs and skip the refinement stage | existing | [pm decision D9](../pm/decisions.md#d9-do-incidents-and-changes-skip-refinement) and [Definition of Ready](../issue-refinement/B-definition-of-ready.md#records-opened-complete) |
