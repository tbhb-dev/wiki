---
type: Design Appendix
title: In-house merge queue architecture and operations
description: The queue's Temporal workflows and worker, its state, failure and recovery, the operator's controls, telemetry, and security debt.
tags: [merge-queue, temporal, operations, observability, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# In-house merge queue architecture and operations

Back to [the design](README.md).

## Components

| Component | Runs on | State | Built by |
| --- | --- | --- | --- |
| Queue workflows | Temporal | Temporal's workflow history | new, merge queue maintainer |
| Queue worker, one Go binary | A container on a service host tbhb-dev owns, under a Unix service account (D7) | A bare mirror per repository, and one SQLite file for history | new, merge queue maintainer |
| Temporal server | First the single-binary server with a SQLite file next to the worker, later the production server ops deploys | Its persistence store | new; the decision to self-host is existing |
| Queue GitHub App | GitHub | Key in 1Password, read through Connect | new, admin VM role under D1 |
| Event gateway | Its own container on an owned host | Its JetStream stream | existing, gateway maintainer |
| CI | Self-hosted runners for private repositories, GitHub-hosted for public ones | Bazel remote cache | existing |
| `tbhbdev alpha merge` and `alpha merge-queue status` | Agent hosts | None | existing and new, tools maintainer |

The deploy follows the ops repository's container pattern. The owner builds the image from the worker repository's main and deploys it without an operator step (existing: deploys need no operator approval).

## Workflows

- **One queue workflow per repository**, with a fixed workflow ID such as `merge-queue/tbhb-dev/ops`. It takes signals for enqueue, dequeue, head change, priority, pause, resume, and main moved. It answers a `status` query with each entry's position and reason code. It continues as new every few hundred events to keep its history short.
- **One group workflow per merge group**, a child of the queue workflow. Its activities build candidates, push branches, wait for CI by signal, set check runs, merge, and verify trees.
- **Activities** do every side effect, each with a retry policy and an idempotency key built from repository, entry, and head SHA. A merge activity treats an already-merged head as success.

The queue's logic lives in plain Go packages behind small interfaces for GitHub, Git, and storage, so the workflows stay thin and the logic is testable without Temporal.

## Intake

- **Target.** The gateway's planned Temporal target passes an event reference, and the workflow reads the full event through the read API by sequence (existing plan: the gateway's D15). Signals go to the repository's queue workflow by its fixed ID.
- **Bridge until then.** The worker reads `GET /v1/events?after=N` for `pull_request`, `pull_request_review`, `check_suite`, `check_run`, `workflow_run`, and `push` events, and signals the workflows. It saves `last_seq` after each page (existing API). A `gap` in the response, meaning retention dropped events, triggers a full reconcile.
- **Reconcile.** At startup, after a gap, and every 15 minutes (D6), a scheduled workflow reads each queue repository's open pull requests with the `merge-queue` label, the candidate heads, and main's tip. It's one paged GraphQL read per repository, and it signals any difference.

## State

Temporal's history holds queue order, groups, and attempts. Pauses live in two places: the queue workflow's state, and a `pauses` table in the worker's SQLite file that the pause activity writes before the workflow acts. GitHub stays the source of truth for labels, heads, approvals, and checks.

The worker's SQLite file holds what outlives a workflow and needs querying:

| Table | Holds |
| --- | --- |
| `proven_trees` | repository, tree SHA, workflow, conclusion, time |
| `runs` | candidate SHA, workflow run ID, conclusion, job and step timings |
| `flakes` | repository, target or job, kind, time |
| `audit` | every GitHub write, with actor and reason |
| `pauses` | each pause and resume per repository, with actor, reason, and time |

Losing that file loses history and the proven-tree shortcut, but not correctness. It moves to the shared Postgres server once that exists (existing plan: [the Postgres design](../postgres/README.md)).

## Failure and recovery

| Failure | Effect | Recovery |
| --- | --- | --- |
| Worker crash | Activities stop | Temporal retries them on restart. Workflows resume from history. The first step of a resumed group flips any `success` check on an unmerged head back to `in_progress` |
| Temporal server down | Workflows stop | They resume from persisted history on restart |
| Temporal data lost | Queue order and in-flight groups gone | The reconcile workflow rebuilds the queues from labels in label-time order, flips stale `success` checks back first, and starts every rebuilt queue paused. It restores pauses from the `pauses` table. A repository with no pause there still waits for a role's resume, and if the table is gone too, only the operator resumes |
| Gateway down | No events | The 15-minute reconcile keeps the queue moving, slower |
| GitHub API errors or rate limits | Writes fail | Activity retries with backoff. Repeated failure pauses the repository with reason `github_unavailable` |
| main moved outside the queue | A built group's base is stale | The `push` signal, or the check before each merge, discards candidates on the old base and rebuilds |
| Tree mismatch after merge | main may hold an untested combination | Pause the repository, alert, and leave main as it is for a person or agent to judge |
| CI never reports | The group waits | The CI timeout counts it as an infrastructure failure: one retry, then removal |
| Host loss | No pull request merges | Redeploy the worker and server on another service host |

## The operator's controls

- **Pause and resume.** `merge-queue pause <repo|all> --reason TEXT` and `merge-queue resume` send signals, and so does the Temporal UI. A pause stops new groups and merges. It doesn't cancel running CI, whose results wait.
- **Who may resume.** A pause records its actor. The command refuses to lift a pause marked as the operator's for anyone else, though the actor isn't authenticated yet (security debt below). The operator's own authoritative stop is the App suspension, which needs their GitHub identity. Roles pause freely, since a pause only stops work.
- **Visibility without help.** The Grafana dashboard, the Temporal UI on the private network, and the `merge-queue` check run on each pull request show queue state, pauses, and history.

### The operator's stop

Two steps need nothing from the worker, Temporal, any agent, or the host:

1. **Suspend the App installation** in the org's GitHub settings. GitHub then refuses every token the App has, so the queue can't push, set checks, or merge.
2. **Merge by hand, if needed.** Remove only the `merge-queue` entry from the repository's main ruleset, then merge as today. Setting the whole ruleset to evaluate isn't an option: that mode needs Enterprise, and it would also drop signature and review requirements (per the reviewer's reading of GitHub's ruleset API docs).

A runbook in `runbooks/` holds these steps once the queue is live (new).

## Telemetry

- **Logs.** JSON lines on stdout from the worker, one per state change and per GitHub write, with repository, number, group, and reason code. Alloy ships them to Loki, as every service must (existing rule). The Temporal server's logs go the same way.
- **Metrics.** The worker's Prometheus endpoint and Temporal's SDK metrics, scraped into Mimir: queue depth, wait time, group duration, step durations, removals by reason, flakes, merges per hour, pauses, and a heartbeat ([appendix D](D-checks-and-flakes.md#timing)).
- **Traces.** OpenTelemetry spans per group and activity into Tempo, through the Temporal SDK's tracing interceptor.
- **Alerts.** Grafana alerts for a stale heartbeat, a tree mismatch, a repository paused over 1 hour, and an entry waiting more than twice its repository's p90 wait.

## Security debt

These go into the security debt tracker rather than blocking the build:

- The App's key reaches the host through Connect, and the worker holds a short-lived token in memory. Anyone with root on that host can merge in every queue repository.
- The App has contents write on every org repository, and workflows write (D17). It can push to any branch except main, which the main ruleset still blocks.
- Once plan step 5 adds the environment policy (D3), candidates run approved pull request code with the main cache token, as `merge_group` runs already do. A malicious approved change could write poisoned cache entries.
- The stale-success window in [appendix E](E-enforcement.md#e1-in-detail) lasts from a crash until the group resumes.
- The single-binary Temporal server and its UI have no authentication and trust the private network. Neither the UI nor the pause command can prove who sent a pause or resume, so the operator-only resume rule is best effort until they authenticate callers.
- One worker, one server, no failover.
