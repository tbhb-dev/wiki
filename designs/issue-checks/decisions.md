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
- **Why:** GitHub's Board layout and built-in project workflows only bind to the built-in `Status` field ([appendix A](A-evidence.md#github-platform-facts)). Turning off the default close and merge workflows keeps unverified closes from bypassing the check gate while preserving native board columns.
- **If yes:** The organization administrator sets those six options on the work board and its calibration twin, turns off the default close and merge workflows, and keeps draft-status values (`status:unverified` and `status:needs-info`) on issue labels instead of board columns.
- **If no:** The board either keeps GitHub's five default options without a `Refinement` column or adds a custom single-select field that built-in board views and project workflows can't drive.
- **Reversible:** Yes. An administrator can rename or reorder `Status` options and toggle built-in workflows in project settings without losing items.

### D2. May the administrator create a dedicated GitHub app for issue checks?

- **Recommendation:** Yes, create `tbhb-dev-issue-checks` with `issues: write`, `projects: write`, and `metadata: read`, installed on repositories with Issues enabled and holding its private key in a default-branch environment.
- **Why:** If the validation workflow used the shared agent account, any agent session could apply `check:passing` to its own issue or edit the findings comment. A dedicated app separates the checker's identity from the author's identity and lets the workflow strip unauthorized `check:*` label edits.
- **If yes:** The organization administrator creates the app, stores its client ID and private key in 1Password and the `issue-checks` environment, and restricts environment deployments to `main`.
- **If no:** The workflow uses each repository's default `GITHUB_TOKEN`, which cannot write to organization Projects v2 boards and cannot cross repositories to verify parent issue types in private twins.
- **Reversible:** Yes. Revoking the key and uninstalling the app removes its access while leaving existing labels and comments intact.

### D3. May the event gateway trigger board-move reverts on project item webhooks?

- **Recommendation:** Yes, add `projects_v2_item` to the organization webhook and let `tbhb-dev/event-gateway` trigger the board guard to revert unauthorized moves into `Ready`, `In progress`, `In review`, or `Done`.
- **Why:** GitHub Actions does not support `projects_v2_item` as a workflow trigger ([appendix A](A-evidence.md#github-platform-facts)), so board moves only reach automation through an organization webhook or polling. Routing the event through the gateway reverts an invalid move in seconds rather than leaving an unready item in `Ready` until a scheduled cron run.
- **If yes:** In Stage 1, the gateway forwards matching `projects_v2_item` edits via `repository_dispatch` to the issue-checks workflow, so the gateway holds no Project write token ([event-gateway D17](../event-gateway/decisions.md#d17-may-the-gateway-hold-a-credential-that-reads-org-project-fields-so-rules-can-match-a-requests-handler)). Once Temporal workers land, the worker uses the checks app credential to revert invalid moves directly.
- **If no:** Only the scheduled sync job (every 15 to 30 minutes) reverts invalid board moves, leaving a longer window where an unready card sits in `Ready`.
- **Reversible:** Yes. Removing `projects_v2_item` from the webhook or disabling the gateway route falls back to the scheduled sync job.

### D4. Should the command-line tool require a draft flag and block builder runs on failing issues?

- **Recommendation:** Yes, require `--draft` on `tbhbdev alpha issue create` and `edit` when a body fails local schema or Vale checks, while always refusing secret or private-detail hits, and make `tbhbdev alpha agent exec` refuse builder runs unless the issue has a `check:passing` result matching the live issue body and no `status:unverified` or `status:needs-info` label.
- **Why:** Requiring `--draft` catches accidental heading or prefix typos locally for callers who meant to file a complete issue, while letting anyone intentionally file an incomplete draft. Stopping builder runs at `agent exec` when the label is missing or stale for the live body guarantees that a brief board race or a web edit never starts code work on a failing or unverified issue.
- **If yes:** The tools role adds `--draft` to `issue create` and `issue edit` in step 3 and enables the fail-closed label and body-digest gate in `agent exec` in step 5 after the backfill completes, exempting `--role refiner` runs so refiners can fix failing drafts.
- **If no:** Filing without `--draft` silently uploads broken bodies that immediately turn `check:failing` on GitHub, or builder runs rely solely on board column position.
- **Reversible:** Yes. A two-line flag change in `tbhb-dev/tools` can switch `--draft` to the default or adjust the `agent exec` check.

### D5. Should the tracker import keep the staging gate for schema and prose checks?

- **Recommendation:** Yes, keep the staging gate so imported items pass Configure Unify Execute (CUE) and Vale checks before creation, and map front-matter `status: Unverified` and `status: Needs info` on the 247 drafts that carry them to `status:unverified` and `status:needs-info` labels at creation.
- **Why:** Filing all 784 staged items as raw drafts would flood the board with drops, duplicates, and unsplit items, trigger 784 validation runs and hundreds of failure comments, and cost roughly 2,000 follow-up API edits under the 200-writes-per-hour throttle. Vetting shape and prose in staging lets every created issue arrive `check:passing` with zero failure comments while the `status:*` labels hold the 247 uncertain items out of `Ready`.
- **If yes:** The imports role keeps [the tracker import procedure](../pm/E-import.md), maps the front-matter `status:` field to `status:unverified` and `status:needs-info`, and allows the Defect reproduction placeholder on `status:needs-info` items.
- **If no:** The importer files all 784 staged items with `issue create --draft`, and refinement runs must split, drop, and fix them through live GitHub API calls.
- **Reversible:** Yes, before the production import runs, by changing the importer flag and batch brief.

## Decided

### D10. How does the check workflow prevent loops and edit races?

- **Decided:** 2026-11-25, by the `pm` role, as a two-way door.
- **Rule:** The check workflow exits in its `if:` guard whenever `github.event.sender.login` is `tbhb-dev-issue-checks[bot]`, runs under the concurrency group `issue-check-${{ github.repository }}-${{ github.event.issue.number }}` with `cancel-in-progress: true`, and compares the live issue body's Secure Hash Algorithm (SHA) 256-bit digest before writing labels or updating the findings comment.
- **Undo:** Two-way door. Change the workflow guard or concurrency settings in `tbhb-dev/rules`.
- **Record:** This design pull request.

### D9. Which check labels and comment layout represent validation state?

- **Decided:** 2026-11-25, by the `pm` role, as a two-way door.
- **Rule:** Validation uses three mutually exclusive check state labels (`check:unchecked`, `check:failing`, and `check:passing`) rather than per-rule labels. Detailed section, rule, limit, actual, and hint findings go into one comment tagged `<!-- tbhb-dev:issue-checks:v1 -->`, created on the first failure and updated in place on later runs.
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
