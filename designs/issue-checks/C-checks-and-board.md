---
type: Design Appendix
title: Checks, board gates, and import
description: Detailed mechanics for the check workflow, the findings comment, board move reverts, command-line stops, and the 784-item tracker import.
tags: [issues, checks, projects, import, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Checks, board gates, and import

Back to [the design](README.md).

Everything in this appendix specifies the target behavior proposed by this design. Where a piece already exists in `tbhb-dev/tools`, `tbhb-dev/pm`, or `tbhb-dev/event-gateway`, the text cites [appendix A](A-evidence.md).

## Scope across repositories and issue types

The check workflow and label gates apply to every public and private repository in the organization where GitHub Issues are enabled:

- **The central work repository and its private twin** hold Initiatives, Epics, and Features ([pm routing](../pm/B-routing-and-taxonomy.md#routing-order-for-create)).
- **Responsible repositories and their private twins** hold Tasks, Defects, Chores, Spikes, and Research issues.
- **The calibration repository** holds the 20 import calibration issues on `work-calibration` ([tracker import calibration](../pm/E-import.md#calibration)).
- **The private service-management records repository** holds Problems, Requests, Decisions, Incidents, and Changes. Problems, Requests, and Decisions use their own Configure Unify Execute (CUE) variants and readiness gates. Under [pm decision D9](../pm/decisions.md#d9-do-incidents-and-changes-skip-refinement), Incidents and Changes open complete through their verbs and skip the `Refinement` board stage, while the check workflow still verifies their schema on open and edit.
- **Repositories with Issues turned off** (`has_issues: false`, such as pure mirror or configuration repositories that route work elsewhere) don't install the caller workflow and carry no `check:*` labels.

## What validation runs and what it costs

The reusable workflow `.github/workflows/issue-checks.yml` in `tbhb-dev/rules` runs the exact headless check that `tbhbdev` runs locally (`tbhbdev content vet-issue`):

1. **Read schema pins.** The job loads `schemas/core.cue`, `schemas/issue.cue`, and `schemas/taxonomy.cue` from the pinned `tbhb-dev/tools` wheel, together with `schemas/issue-sections.cue` and `schemas/issue.<variant>.cue` from the pinned `tbhb-dev/pm` commit (or the service variant schema passed by the private records repository).
2. **Normalize.** `tbhbdev.content_model.normalize` parses the live issue title, body, issue type, `area/` and `role/` labels, parent link, and organization issue fields into the JSON block model.
3. **Evaluate CUE.** `cue vet --all-errors` checks section order, required sections, word and character caps, and one-based prefixed list items (`AC1:`, `V1:`, `S1:`). It also checks that `Out of scope` is present (listing exclusions or a single `None.` item, except on the temporary `imported` variant before retirement).
4. **Run kernel and relation checks.** Using the checks app token, the runner fetches the parent issue (if any) to verify parent type rules (`Initiative` for an `Epic`, `Epic` for a `Feature`, non-`Task` for a `Task`, `Problem` or `Epic` for a `Defect` with a parent) and public-to-private visibility rules. It also checks for links-only sections, note flooding, and title shape warnings.
5. **Run Vale and scans.** The runner runs `vale --ext=.md` with the organization's `.vale.ini` and styles from `tbhb-dev/rules`, applying the short-body rule that downgrades errors to warnings when prose is under half the variant's word cap. Finally, it runs the gitleaks secret scan and private-detail scan on the title and body.

Each run executes on a standard Linux runner with zero large language model (LLM) calls and zero paid external services. It makes 2 to 4 GitHub API requests and finishes in 8 to 15 seconds.

## Labels and loop prevention

### Who sets the labels and how tampering is stopped

Only the `tbhb-dev-issue-checks` GitHub App holds the credential used by the check workflow. Because GitHub has no per-label access control list, any account with `issues: write` could call the REST API directly. Two controls stop manual or agent tampering:

1. **Command-line and kernel refusal.** `tbhbdev.content` and `tbhbdev alpha issue` reject any `--label` argument starting with `check:`. Only the tracker importer and `tbhbdev pm refine` may pass `status:unverified` or `status:needs-info`. Agent harness rules deny raw `gh issue edit` label commands.
2. **Webhook revert.** When an `issues` event with action `labeled` or `unlabeled` arrives from any `sender.login` other than `tbhb-dev-issue-checks[bot]` and touches a `check:*` or `status:*` label, the workflow re-evaluates the issue and restores the authoritative label set.

### Loop prevention and edit races

- **Sender guard.** The workflow's job-level `if:` condition skips any `issues` or `issue_comment` event where `github.event.sender.login == 'tbhb-dev-issue-checks[bot]'`. When the checks app swaps `check:failing` for `check:passing` or updates the findings comment, that write never triggers a second run.
- **Label filter.** For non-bot senders on `labeled` and `unlabeled`, the workflow runs only when `github.event.label.name` starts with `check:`, `status:`, `area/`, or `role/`.
- **Concurrency and Secure Hash Algorithm (SHA) guard.** Each run joins the concurrency group `issue-check-${{ github.repository }}-${{ github.event.issue.number }}` with `cancel-in-progress: true`. Before writing labels or patching the comment, the runner re-reads the live issue body and compares its `SHA-256` digest with the digest it just validated; on a mismatch, it exits cleanly so the newer run writes the result.
- **Edit-storm protection.** If a single issue triggers more than 5 check runs within 10 minutes, the workflow logs a rate warning and `tbhb-dev/event-gateway` emits an alert so the operator or coordinator can stop a runaway loop.

### The single in-place findings comment

The checks app maintains at most one comment per issue, looked up by the stored comment's author (`user.login == 'tbhb-dev-issue-checks[bot]'`, the app's bot login rather than the current webhook event's `sender.login`) and the hidden marker `<!-- tbhb-dev:issue-checks:v1 -->` in the comment body:

```markdown
<!-- tbhb-dev:issue-checks:v1 -->
### Issue checks: failing

Evaluated variant `feature` at body `sha256:9f4c...` on 2026-11-25T14:20:00Z. Fix the refusals below by editing the issue body.

| Kind | Section | Rule | Limit | Actual | Hint |
| --- | --- | --- | --- | --- | --- |
| Refusal | Acceptance criteria | prefixed-list-id | `^AC1: \S` | `- First check.` | Start each bullet with `AC1: ` and count up by one. |
| Refusal | Out of scope | required-section | present | missing | List deliberate exclusions, or write `- None.` after considering scope. |
| Warning | Title | title-shape | `<Role> can <action>` | `Add check labels` | Use a role and what it can do. |
```

If an issue passes validation on its first `opened` event, the app applies `check:passing` and creates no comment. Once a comment exists because an earlier revision failed or a board move was reverted, a later edit that fixes every refusal switches the label to `check:passing` and updates that same comment in place to `### Issue checks: passing` with the passing body's `SHA-256` digest. If a web-UI edit in a public repository trips a secret or private-detail scan rule, the table prints only the rule ID and line number and never quotes the offending text.

### Existing issues

Open issues created before the workflow ships carry no `check:*` label, so a `workflow_dispatch` backfill mode on `issue-checks.yml` runs in rollout step 4 under the 200-writes-per-hour budget to evaluate each open issue, set `check:passing` or `check:failing`, and post the findings comment only on failing items. Running the validator and backfill before rollout step 5 enables the fail-closed `tbhbdev alpha agent exec` gate ensures existing valid issues receive `check:passing` without interrupting ordinary builds. Once the launcher gate is active, `tbhbdev alpha agent exec` treats any unlabeled open issue (such as a newly reopened issue awaiting its check run) as `check:unchecked` and refuses builder runs.

## Board enforcement and command-line gates

### How a board revert works and what stays unenforced

When someone moves an item on the work board, GitHub sends a `projects_v2_item` (`action: edited`) webhook to the organization webhook receiver in `tbhb-dev/event-gateway`. The payload's `changes.field_value` object gives the field name (`Status`) and the `from` and `to` options.

1. If `to` is `Ready`, `In progress`, `In review`, or `Done` and `sender.login` is not `tbhb-dev-issue-checks[bot]`, the gateway triggers the board guard (via `repository_dispatch` to `issue-checks.yml` in Stage 1, or a Temporal activity holding the checks app credential once Temporal lands). Moves where `to` is `Backlog` or `Refinement` are not intercepted, so any project writer can return an item from any stage to `Backlog` or `Refinement`.
2. The guard reads the issue's labels, its `Out of scope` section, its linked pull requests, and its readiness record, accepting either a current `ready` refinement record comment on the issue or an approved staging record on a tracker-import batch Chore ([status transitions](../issue-refinement/D-status-transitions.md#current-records)). When the issue has no newer refinement comment or queue return, the guard finds the staging record by scanning the issue's timeline for a `cross-referenced` event from a batch Chore sub-issue under the import Epic, confirming the Chore has a passing batch-review verdict, and matching the Chore's recorded issue link and vetted body `SHA-256` digest against the live issue.
3. If the `from` stage or any target gate condition in [the README transition table](README.md#board-transitions-and-enforcement) fails, the guard calls `updateProjectV2ItemFieldValue` to revert `Status` to the prior stage (or `Refinement` when `from` was empty or invalid) and updates the `<!-- tbhb-dev:issue-checks:v1 -->` comment with a `Board gate` notice naming the reverted target stage and the missing requirement.
4. **After-the-fact revert latency and what stays unenforced.** GitHub Projects v2 offers no synchronous pre-move hook (unverified), so an invalid drag briefly enters `Ready` before the guard reverts it after the fact in an estimated 2 to 5 seconds from a direct gateway worker or 10 to 30 seconds via Stage 1 `repository_dispatch` (unverified). While the card sits in `Ready` during that window, the `check:*` and `status:*` labels and the `tbhbdev alpha agent exec` launcher gate still hold, and `DraftIssue` cards with no repository issue always revert to `Backlog`. Beyond that revert window, the board guard leaves moves into `Backlog` and `Refinement`, mover identity on otherwise-valid transitions, card ordering within a column, and non-`Status` project fields unenforced.

### Command-line behavior in tools

- **`tbhbdev alpha issue create` and `tbhbdev alpha issue edit`:** Both commands render template Markdown when given `--template <variant>` and section values, or read `--body-file`. Both always run the secret scan, the private-detail routing check, and the agent session trailer check first, refusing any violation immediately. Without `--draft`, both run `tbhbdev.content_vet.check` and Vale locally and exit non-zero on any refusal. With `--draft`, schema, kernel-list, and Vale refusals print to standard error as warnings, and the command writes the issue with `check:unchecked` so the workflow evaluates it and posts the findings comment.
- **`tbhbdev alpha agent exec`:** When invoked for a build run (`--profile build` or `--role builder`) that references an issue, the launcher queries the issue's live labels, body, comments, and timeline before creating a worktree. It refuses to start if `check:passing` is absent, if `status:unverified` or `status:needs-info` is present, or if the passing result does not cover the live issue body (verifying that the live body's `SHA-256` digest matches the `### Issue checks: passing` comment when present and the current `ready` refinement or batch Chore staging record, and refusing any issue whose last body edit or check comment is newer than the `check:passing` label event). Runs launched with `--role refiner` bypass this stop so refiners can edit and verify failing drafts.

### Cross-role pieces and build order

| Order | Piece | Owner | Repository |
| --- | --- | --- | --- |
| 1 | `tbhb-dev-issue-checks` GitHub App, `issue-checks` environment, five check/status labels, and work board `Status` options (`Backlog`, `Refinement`, `Ready`, `In progress`, `In review`, `Done`) with built-in close and merge workflows turned off | Organization administrator | Organization settings |
| 2 | Tagged CUE variant schemas and templates with required `Out of scope` (`None.` allowed) and prefixed list IDs (`AC1:`, `V1:`, `S1:`) | `pm` | `tbhb-dev/pm` |
| 3 | Headless `content vet-issue` entry point, template rendering, `alpha issue edit`, `--draft` on `create` and `edit`, and `check:*` label reservation | tools | `tbhb-dev/tools` |
| 4 | Service-management variant alignment for Problems, Requests, and Decisions, plus `Incident` and `Change` stage exemptions | `sm` | Private records repository |
| 5 | Reusable `.github/workflows/issue-checks.yml` workflow, caller workflows in issue-enabled repositories, and one-time backfill run across existing open issues | Rules maintainer and `pm` | `tbhb-dev/rules` and issue repositories |
| 6 | Organization webhook subscription for `projects_v2_item`, gateway route to the board guard, and fail-closed `alpha agent exec` check-label and body-digest gate | Gateway maintainer and tools | `tbhb-dev/event-gateway` and `tbhb-dev/tools` |

## How the 784-item tracker import uses the checks

The staged tracker import holds 784 items across Chores, Defects, Features, Problems, Tasks, Epics, Research, and Spikes. Across those eight types, 247 drafts carry front-matter `status: Unverified` or `status: Needs info`.

1. **Keep the staging gate for text shape and prose.** Under [D5](decisions.md#d5-should-the-tracker-import-keep-the-staging-gate-for-schema-and-prose-checks), the imports role continues to resolve drops and splits in staging and vets every surviving file against its destination CUE schema and the Vale pass before creating an issue ([tracker import](../pm/E-import.md#staging-and-refinement)). For a Defect with `status: Needs info` whose reproduction steps are still unknown, the schema accepts the exact placeholder paragraph `Find a reproduction and add it here.` ([Definition of Ready](../issue-refinement/B-definition-of-ready.md#no-procedure)).
2. **Map front-matter `status:` to the checks label set.** When the importer creates each issue in batches of 25 under the 200-writes-per-hour budget, it strips `status:` from the frontmatter and applies `status:unverified` (for `status: Unverified`) or `status:needs-info` (for `status: Needs info`), as settled in [D7](decisions.md#d7-where-do-the-front-matter-status-values-in-the-imported-drafts-go).
3. **Effect on the board, notifications, and agent work.** Because each created issue already satisfies its CUE variant and Vale rules, the `opened` run of `issue-checks.yml` applies `check:passing` and posts zero failure comments, avoiding 784 comment notifications. Verified items with a passing batch review enter `Ready`. The 247 items carrying `status:unverified` or `status:needs-info` land in `Backlog` or `Refinement` with `check:passing` plus their `status:*` label, where the board guard reverts any premature move into `Ready` after the fact and `tbhbdev alpha agent exec` blocks builder runs until a refiner verifies the claim or resolves the question and clears the label.
