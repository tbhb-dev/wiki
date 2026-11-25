---
type: Design Appendix
title: Status checks for issues alternatives
description: The options not taken for check labels, board enforcement, command-line draft filing, and the tracker import.
tags: [issues, checks, alternatives, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Status checks for issues alternatives

Back to [the design](README.md).

## Label set and validation identity

### Per-check detail labels on the issue

The workflow could apply one label per failing rule class, such as `check:schema-fail`, `check:vale-fail`, `check:parent-fail`, and `check:scope-fail`. That approach clutters the issue sidebar, fires multiple `labeled` and `unlabeled` webhook deliveries on every edit, and still can't show line numbers, word counts, or fix hints. Keeping three mutually exclusive check state labels (`check:unchecked`, `check:failing`, and `check:passing`) plus the two draft-status labels (`status:unverified` and `status:needs-info`) gives clean filters for board views and `tbhbdev`, while the single in-place comment holds the per-section table.

### GitHub checks API on issues

Pull requests use the GitHub Checks API to attach check runs to a commit Secure Hash Algorithm (SHA). GitHub's Checks API binds check suites and check runs only to commits and pull requests, never to issues. Labels managed by a dedicated GitHub app are the closest issue primitive that works in repository filters, Project board queries, webhook payloads, and the command-line tool.

### Storing check status in an organization issue field

The check result could live in an organization issue field instead of a `check:*` label. Standard `issues` webhook payloads include the issue's full label list on every event, whereas organization issue field values require an extra GraphQL query and an organization-field read grant that `tbhb-dev/event-gateway` doesn't hold ([event-gateway D17](../event-gateway/decisions.md#d17-may-the-gateway-hold-a-credential-that-reads-org-project-fields-so-rules-can-match-a-requests-handler)). Labels let the gateway, the launcher, and board filters inspect check state directly from the issue payload.

### Using the default workflow token or the shared agent account

Running the validator under `GITHUB_TOKEN` or the shared agent account avoids creating a new GitHub App. However, `GITHUB_TOKEN` is scoped to one repository, so it can't read a parent issue's type in a private twin or revert a status move on an organization Project board. Using the shared agent account would give every agent session the same identity as the validator, allowing an agent to set `check:passing` on its own issue. A dedicated `tbhb-dev-issue-checks` app isolates the authority to set check labels and revert board moves.

## Board fields and move enforcement

### Custom status field instead of the built-in status field

A custom single-select Project field or an organization issue field could hold the six stages (`Backlog`, `Refinement`, `Ready`, `In progress`, `In review`, `Done`). In GitHub Projects v2, the Board layout defaults to the built-in `Status` field, and built-in project workflows only update the built-in `Status` field ([appendix A](A-evidence.md#github-platform-facts)). Using a custom field leaves the built-in `Status` field as a redundant column on every project. Configuring the six options directly on the built-in `Status` field and turning off the default close and merge workflows keeps one status column per board.

### Draft-status values as board columns

`Unverified` and `Needs info` could appear as extra columns on the work board beside `Backlog` and `Refinement`. That mixes workflow stages with orthogonal facts about why an item isn't ready, and an item can be both `Unverified` and failing schema checks at the same time. By operator direction ([D7](decisions.md#d7-where-do-the-front-matter-status-values-in-the-imported-drafts-go)), `status:unverified` and `status:needs-info` live in the checks label set while the item sits in `Backlog` or `Refinement`.

### Scheduled polling alone for board moves

A scheduled GitHub Actions cron job running every 15 to 30 minutes could scan the board and revert invalid moves without touching `tbhb-dev/event-gateway`. During the 30-minute gap, an unready issue would appear in `Ready` to human readers and queries. Routing `projects_v2_item` webhooks through the event gateway cuts revert latency to a few seconds while keeping the scheduled sync job as a fallback sweep.

### Holding a separate gate field instead of reverting status

Instead of reverting an invalid drag into `Ready`, the workflow could leave `Status` at `Ready` and set a second `Gate: Blocked` field. Every consumer and board view would then have to join two fields to know whether an item is actually ready, and the `Ready` column on the board would contain blocked drafts. Reverting `Status` to `Refinement` (or the prior stage) and explaining why in the findings comment keeps the `Ready` column honest.

## Command-line draft filing and the tracker import

### Filing drafts by default without a flag

`tbhbdev alpha issue create` and `edit` could file a draft whenever local Configure Unify Execute (CUE) or Vale validation fails, without requiring `--draft`. Under that default, an agent that makes a minor heading typo or forgets a `V1:` prefix silently creates a `check:failing` issue on GitHub and triggers an Actions run instead of fixing the text locally in one second. Keeping local validation as the default and adding `--draft` for deliberate draft filing preserves fast local feedback.

### Filing all 784 imported items as raw drafts

The 784-item tracker import could skip the staging gate in [the tracker import design](../pm/E-import.md) and pass every file straight to `tbhbdev alpha issue create --draft`. That option puts dropped items, duplicates, and unsplit multi-change items onto the production board, triggers 784 check workflow runs and hundreds of failure comments on day one, and forces refinement agents to fix every prose and section error through throttled GitHub API calls. Keeping the staging gate for CUE and Vale checks while mapping the 247 front-matter `status:` values to `status:unverified` and `status:needs-info` lets every created issue arrive `check:passing` without noise.
