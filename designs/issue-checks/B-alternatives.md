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

Pull requests use the GitHub Checks API to attach check runs to a commit Secure Hash Algorithm (SHA). GitHub's Checks API binds check suites and check runs only to commits and pull requests, never to issues. Labels written by a dedicated GitHub app are the closest issue primitive that works in repository filters, Project board queries, webhook payloads, and the command-line tool.

### Storing check status in an organization issue field

The check result could live in an organization issue field instead of a `check:*` label. Standard `issues` webhook payloads include the issue's full label list on every event, whereas organization issue field values require an extra GraphQL query and an organization-field read grant that `tbhb-dev/event-gateway` doesn't hold ([event-gateway D17](../event-gateway/decisions.md#d17-may-the-gateway-hold-a-credential-that-reads-org-project-fields-so-rules-can-match-a-requests-handler)). Labels let the gateway, the launcher, and board filters inspect check state directly from the issue payload.

### Using the default workflow token or the shared agent account

Running the validator under `GITHUB_TOKEN` or the shared agent account avoids creating a new GitHub App. However, `GITHUB_TOKEN` is scoped to one repository, so it can't read a parent issue's type in a private twin or revert a status move on an organization Project board. Using the shared agent account would give every agent session the same identity as the validator, allowing an agent to set `check:passing` on its own issue. A dedicated `tbhb-dev-issue-checks` app isolates the authority to write check labels and the board `Status` field.

### Letting the command-line tool write the unchecked label

The draft-filing path could apply `check:unchecked` itself when it files a draft. That gives the `check:*` labels a second writer on the shared agent account, which the label refusal in the same tool exists to prevent. The checks app sets `check:unchecked` on the first run instead, and every gate treats an issue with no check label as unchecked.

### Restoring a draft-status label from the checks app

On a manual change to `status:unverified` or `status:needs-info`, the checks app could write the label back. That makes the app a second writer of the draft-status labels, and the event sender can't tell the kernel's importer or refiner from another session on the same account. The run compares the label with the issue's current kernel record instead and fails the check on a mismatch, so the kernel stays the only writer.

### Separate freshness signals for the body and the result

The launcher could compare a body-only hash, the time of the `check:passing` label event, or the findings comment's edit time. A title-only edit changes none of the first two, and a board guard message changes the third without any change to the issue. One input digest over the title, the body, and every other validated input replaces all three.

### Other ways to check again after the last filing write

The kernel could send a `workflow_dispatch` event after its last filing write, which needs a new Actions write grant for the kernel's account on every issue repository. The validator could instead treat late-written fields as pending for a short window, which guesses at timing and can still mark a valid issue failing. Applying labels as the last filing write uses a write the kernel already makes ([D11](decisions.md#d11-how-does-an-issue-get-checked-again-after-the-kernels-last-filing-write)).

## Board fields and move enforcement

### Custom status field instead of the built-in status field

A custom single-select Project field or an organization issue field could hold the six stages (`Backlog`, `Refinement`, `Ready`, `In progress`, `In review`, `Done`). In GitHub Projects v2, the Board layout defaults to the built-in `Status` field, and built-in project workflows only update the built-in `Status` field ([appendix A](A-evidence.md#github-platform-facts)). Using a custom field leaves the built-in `Status` field as a redundant column on every project. Configuring the six options directly on the built-in `Status` field and turning off the default close and merge workflows keeps one status column per board.

### Draft-status values as board columns

`Unverified` and `Needs info` could appear as extra columns on the work board beside `Backlog` and `Refinement`. That mixes workflow stages with orthogonal facts about why an item isn't ready, and an item can be both `Unverified` and failing schema checks at the same time. By operator direction ([D7](decisions.md#d7-where-do-the-front-matter-status-values-in-the-imported-drafts-go)), `status:unverified` and `status:needs-info` live in the checks label set while the item sits in `Backlog` or `Refinement`.

### Scheduled polling alone for board moves

The check workflow's scheduled sweep, running every 15 to 30 minutes, could be the board guard's only trigger besides check runs, without touching `tbhb-dev/event-gateway`. During the 30-minute gap, an unready issue would appear in `Ready` to human readers and queries. Routing `projects_v2_item` webhooks through the event gateway cuts the estimated latency to 10 to 30 seconds (unverified) while keeping the check workflow's scheduled sweep as a fallback.

### Holding a separate gate field instead of reverting status

Instead of reverting an invalid drag into `Ready`, the workflow could leave `Status` at `Ready` and set a second `Gate: Blocked` field. Every consumer and board view would then have to join two fields to know whether an item is actually ready, and the `Ready` column on the board would contain blocked drafts. Moving the card to the stage its issue qualifies for, or to `Refinement`, and explaining why in the `Board gate` section of the findings comment keeps the `Ready` column honest.

### Reverting from the event payload

The guard could revert a card to the `from` value in the `projects_v2_item` payload. A delayed run would then overwrite a later valid move, or restore a stage the issue no longer qualifies for. The guard reads the live column and the live issue before every write instead ([D12](decisions.md#d12-which-component-writes-the-board-status-field-and-how-does-it-avoid-stale-writes)).

### Letting the sync job advance cards

The sync job could write the board `Status` field as well as the organization issue field. The board would then have two automated writers with separate credentials, and a revert could race an advance. The board guard is the one automated writer of the board field, and the sync job keeps its one field.

### A second comment for board guard messages

The guard could post its own comment when it rejects a move. Each rejected move would then add a comment, and a reader would need to join two comments to see an issue's state. Guard messages go in the `Board gate` section of the one findings comment and never change the check record.

## Command-line draft filing and the tracker import

### Filing drafts by default without a flag

`tbhbdev alpha issue create` and `edit` could file a draft whenever local Configure Unify Execute (CUE) or Vale validation fails, without requiring `--draft`. Under that default, an agent that makes a minor heading typo or forgets a `V1:` prefix creates an issue that turns `check:failing` on GitHub instead of fixing the text locally in one second. Keeping local validation as the default and adding `--draft` for deliberate draft filing preserves fast local feedback.

### Filing all 784 imported items as raw drafts

The 784-item tracker import could skip the staging gate in [the tracker import design](../pm/E-import.md) and pass every file straight to `tbhbdev alpha issue create --draft`. That option puts dropped items, duplicates, and unsplit multi-change items onto the production board, produces hundreds of failing findings comments on day one, and forces refinement agents to fix every prose and section error through throttled GitHub API calls. Keeping the staging gate lets each vetted issue arrive `check:passing`, and the 247 front-matter `status:` values map to `status:unverified` and `status:needs-info` under either option.

### Changing the Defect schema to accept a placeholder reproduction

The Defect variant in `tbhb-dev/pm` could accept the placeholder paragraph for a Defect whose reproduction is unknown. That puts an import-only exception into a schema every Defect uses. The import files such a Defect as a draft with `status:needs-info` instead, as [appendix C](C-checks-and-board.md#how-the-784-item-tracker-import-uses-the-checks) describes.
