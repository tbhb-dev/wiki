---
type: Design Appendix
title: Status checks for issues decisions
description: The open and decided questions of the status checks for issues design.
tags: [issues, checks, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Status checks for issues decisions

Back to [the design](README.md).

## Open

### D1. Should the work board use the built-in status field with six stage options?

- **Recommendation:** Yes, configure the built-in `Status` field on the work board with `Backlog`, `Refinement`, `Ready`, `In progress`, `In review`, and `Done`, and turn off GitHub's built-in close and merge workflows.
- **Why:** GitHub's Board layout and built-in project workflows only bind to the built-in `Status` field ([appendix A](A-evidence.md#github-platform-facts)). Turning off the default close and merge workflows leaves the board guard as the field's only automated writer while preserving native board columns.
- **If yes:** The organization administrator sets those six options on the work board and its calibration twin and turns off the default close and merge workflows. The draft-status values stay on the `status:unverified` and `status:needs-info` labels, and the sync job keeps writing only the organization issue field.
- **If no:** The board either keeps GitHub's five default options without a `Refinement` column or adds a custom single-select field that built-in board views and project workflows can't drive. The board guard then writes whichever field holds the stages, under the same rule in [D12](#d12-which-component-writes-the-board-status-field-and-how-does-it-avoid-stale-writes).
- **Reversible:** Yes. An administrator can rename or reorder `Status` options and toggle built-in workflows in project settings without losing items.

### D2. May the administrator create a dedicated GitHub app for issue checks?

- **Recommendation:** Yes, create `tbhb-dev-issue-checks` with `issues: write`, `projects: write`, and `metadata: read`, installed on repositories with Issues enabled and holding its private key in a default-branch environment.
- **Why:** If the validation workflow used the shared agent account, any agent session could apply `check:passing` to its own issue or edit the findings comment. A dedicated app separates the checker's identity from the author's identity, and its bot login is how the workflow and the launcher find the one findings comment.
- **If yes:** The organization administrator creates the app, stores its client ID and private key in 1Password and the `issue-checks` environment, and restricts environment deployments to `main`.
- **If no:** The workflow uses each repository's default `GITHUB_TOKEN`, which cannot write to organization Projects v2 boards and cannot cross repositories to verify parent issue types in private twins.
- **Reversible:** Yes. Revoking the key and uninstalling the app removes its access while leaving existing labels and comments intact.

### D3. May the event gateway trigger the board guard on project item webhooks?

- **Recommendation:** Yes, add `projects_v2_item` to the organization webhook and let `tbhb-dev/event-gateway` dispatch each `Status` edit to the issue-checks workflow, where the board guard runs.
- **Why:** GitHub Actions does not support `projects_v2_item` as a workflow trigger ([appendix A](A-evidence.md#github-platform-facts)), so board moves only reach automation through an organization webhook or polling. The dispatch lets the guard move an invalid card back in an estimated 10 to 30 seconds (unverified) instead of at the next scheduled sweep.
- **If yes:** The gateway forwards each `projects_v2_item` `Status` edit that the checks app didn't send, through `repository_dispatch`, to the issue-checks workflow. The gateway holds no Project write token ([event-gateway D17](../event-gateway/decisions.md#d17-may-the-gateway-hold-a-credential-that-reads-org-project-fields-so-rules-can-match-a-requests-handler)), and the guard writes with the checks app credential.
- **If no:** The guard still runs after every check run and on its scheduled sweep every 15 to 30 minutes. An invalid drag then shows in a gated column until the next of those runs, while the check labels and the launcher gate keep holding.
- **Reversible:** Yes. Removing `projects_v2_item` from the webhook or disabling the gateway route falls back to the check runs and the scheduled sweep.

### D4. Should the command-line tool require an explicit draft flag to file an issue that fails local checks?

- **Recommendation:** Yes, require `--draft` on `tbhbdev alpha issue create` and `edit` when a body fails local schema or Vale checks, while always refusing secret or private-detail hits.
- **Why:** Requiring `--draft` catches accidental heading or prefix typos locally for callers who meant to file a complete issue, while letting anyone intentionally file an incomplete draft. This item covers only the flag. The builder stop is settled in [D6](#d6-may-callers-file-draft-issues-that-fail-schema-validation) and applies under either answer.
- **If yes:** The tools role adds `--draft` to `issue create` and `issue edit` in plan step 3. Without the flag, a body that fails local checks exits non-zero and files nothing.
- **If no:** A body that fails local checks files as a draft with no flag, and the caller learns of a typo only when the check run sets `check:failing`. The launcher still refuses builder runs on that issue.
- **Reversible:** Yes. A small flag change in `tbhb-dev/tools` switches between the two defaults.

### D5. Should the tracker import keep the staging gate for schema and prose checks?

- **Recommendation:** Yes, keep the staging gate. [pm decision D11](../pm/decisions.md#d11-which-issue-types-does-the-tracker-import-create) of 2026-11-23 already has the import vet each staged item before the kernel creates it, and this item asks whether that ruling stands now that [D6](#d6-may-callers-file-draft-issues-that-fail-schema-validation) allows draft filing.
- **Why:** Filing all 784 staged items as raw drafts would put drops, duplicates, and unsplit items on the board, produce hundreds of failing findings comments, and cost roughly 2,000 follow-up API edits under the 200-writes-per-hour throttle. Vetting shape and prose in staging lets each created issue arrive `check:passing`.
- **If yes:** The imports role keeps [the tracker import procedure](../pm/E-import.md). The importer maps front-matter `status: Unverified` and `status: Needs info` on the 247 drafts that carry them to `status:unverified` and `status:needs-info` at filing, as [D7](#d7-where-do-the-front-matter-status-values-in-the-imported-drafts-go) settles. A Defect that still lacks a reproduction is the one exception to the gate: the import files it as a draft with `status:needs-info`, its outcome stays `check:unchecked`, and the import validates it after refinement.
- **If no:** The importer files all 784 staged items through the draft-filing path, and refinement runs split, drop, and fix them through live GitHub API calls. The same front-matter mapping to `status:unverified` and `status:needs-info` applies at filing, and a Defect that lacks a reproduction is handled the same way.
- **Reversible:** Yes, before the production import runs, by changing the importer flag and batch brief.

## Decided

### D13. Who writes each label, and what happens on an unauthorized label change?

- **Decided:** 2026-11-25, by the `pm` role, as a two-way door.
- **Rule:** The checks app is the only writer of `check:unchecked`, `check:failing`, and `check:passing`, and it sets `check:unchecked` itself on the first run, so the command-line tool and the importer never write a `check:*` label. The content kernel is the only writer of `status:unverified` and `status:needs-info`, through its import and refine verbs, and the issue's current kernel record is their authoritative source. A run rewrites a `check:*` label that another sender changed, and reports `draft-status-mismatch` with `check:failing` while a draft-status label disagrees with its record.
- **Undo:** Two-way door. Change the label refusal in `tbhb-dev/tools` and the mismatch check in `tbhb-dev/rules`.
- **Record:** This design pull request.

### D12. Which component writes the board status field, and how does it avoid stale writes?

- **Decided:** 2026-11-25, by the `pm` role, as a two-way door.
- **Rule:** The board guard, a job of the issue-checks workflow that uses the checks app credential, is the only automated writer of the board `Status` field: it advances, moves back, and demotes cards, while the sync job keeps writing only the organization issue field. Before every write the guard reads the card's live column and the issue's live state, and it never acts on the `from` and `to` values of an event. Its messages go in the `Board gate` section of the one findings comment and never change the check result.
- **Undo:** Two-way door. Change the guard job in `tbhb-dev/rules`.
- **Record:** This design pull request.

### D11. How does an issue get checked again after the kernel's last filing write?

- **Decided:** 2026-11-25, by the `pm` role, as a two-way door.
- **Rule:** The kernel filing path applies an issue's labels as its last filing write, after the organization fields, the parent link, and the project item. The `labeled` events start a follow-up run that reads the finished issue, and the scheduled sweep covers an issue filed with no label. This needs no new grant, unlike a `workflow_dispatch` call from the kernel's account.
- **Undo:** Two-way door. Change the write order in `tbhb-dev/tools`.
- **Record:** This design pull request.

### D10. How does the check workflow prevent loops and edit races?

- **Decided:** 2026-11-25, by the `pm` role, as a two-way door.
- **Rule:** The check workflow exits in its `if:` guard whenever `github.event.sender.login` is `tbhb-dev-issue-checks[bot]`, and runs under one concurrency group per repository and issue number with `cancel-in-progress: true`. Before it writes a label or the findings comment, it reads the validated inputs again and compares their Secure Hash Algorithm (SHA) 256-bit digest (`inputs_sha256`) with the digest it validated. That digest covers the title, the body, and every other input the validator reads.
- **Undo:** Two-way door. Change the workflow guard or concurrency settings in `tbhb-dev/rules`.
- **Record:** This design pull request.

### D9. Which check labels and comment layout represent validation state?

- **Decided:** 2026-11-25, by the `pm` role, as a two-way door.
- **Rule:** Validation uses three mutually exclusive check state labels (`check:unchecked`, `check:failing`, and `check:passing`) rather than per-rule labels. Findings go into one comment per issue, tagged `<!-- tbhb-dev:issue-checks:v1 -->`, created once on the first completed run, and updated in place afterward. Readers find it by the checks app's bot login as its author plus the marker, and it starts with the check record that holds the result and `inputs_sha256`.
- **Undo:** Two-way door. Add per-check labels to the label sync and workflow if sidebar filtering by failure kind becomes necessary.
- **Record:** This design pull request.

### D8. What gates an issue moving into the ready stage on the board?

- **Decided:** 2026-11-25, by the operator.
- **Rule:** Moving an issue into `Ready` requires `check:passing`, the absence of both `status:unverified` and `status:needs-info`, a current `ready` refinement record, and a decided `Out of scope` section (either listing specific exclusions or containing a single `None.` item, the sixteenth readiness check of the Definition of Ready).
- **Undo:** Two-way door. Amend the readiness check in [the Definition of Ready](../issue-refinement/B-definition-of-ready.md) and the variant schemas in `tbhb-dev/pm`.
- **Record:** tbhb direction, 2026-11-25, and [the Definition of Ready](../issue-refinement/B-definition-of-ready.md#readiness-checks).

### D7. Where do the front-matter status values in the imported drafts go?

- **Decided:** 2026-11-25, by the operator.
- **Rule:** The front-matter `status:` values in the imported drafts (`Unverified` and `Needs info`, present on 247 of the 784 drafts across Chores, Defects, Features, Problems, Tasks, Epics, Research, and Spikes) become the `status:unverified` and `status:needs-info` labels in the checks label set. They are not a board stage and not a separate field, and the importer maps the front-matter field to these labels when it files each draft.
- **Undo:** Two-way door. Remove the two labels and migrate their state to an issue field or board column.
- **Record:** tbhb direction, 2026-11-25.

### D6. May callers file draft issues that fail schema validation?

- **Decided:** 2026-11-25, by the operator.
- **Rule:** Issue validation stops being all-or-nothing at filing time. A draft issue with missing sections or incomplete information can be filed, a workflow validates it on issue and comment events against `pm`'s CUE schemas and sets status-check labels plus one updated comment listing what fails, and a failing issue cannot start a builder run or stay in `Ready` (any move into `Ready` is reverted after the fact).
- **Undo:** Two-way door. Restore strict synchronous schema refusal on every issue creation and retire the check workflow.
- **Record:** tbhb direction, 2026-11-25.
