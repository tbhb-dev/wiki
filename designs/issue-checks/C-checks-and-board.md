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

The checks app maintains at most one comment per issue, identified by `sender.login == 'tbhb-dev-issue-checks[bot]'` and the hidden marker `<!-- tbhb-dev:issue-checks:v1 -->`:

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

Open issues created before the workflow ships carry no `check:*` label. A `workflow_dispatch` backfill mode on `issue-checks.yml` iterates through open issues under the 200-writes-per-hour budget, evaluates each one, sets `check:passing` or `check:failing`, and posts the findings comment only on failing items. Until the backfill reaches an existing open issue, `tbhbdev alpha agent exec` treats the missing label as `check:unchecked` and refuses builder runs. Closed issues are evaluated only if reopened.

## Board enforcement and command-line gates

### How a board block works and what stays unenforced

When someone moves an item on the work board, GitHub sends a `projects_v2_item` (`action: edited`) webhook to the organization webhook receiver in `tbhb-dev/event-gateway`. The payload's `changes.field_value` object gives the field name (`Status`) and the `from` and `to` options.

1. If `to` is `Ready`, `In progress`, `In review`, or `Done` and `sender.login` is not `tbhb-dev-issue-checks[bot]`, the gateway triggers the board guard (via `repository_dispatch` to `issue-checks.yml` in Stage 1, or a Temporal activity holding the checks app credential once Temporal lands).
2. The guard reads the issue's labels, its latest refinement record comment, its `Out of scope` section, and linked pull requests.
3. If any gate condition from the transition table in [the README](README.md#board-transitions-and-enforcement) fails, the guard executes `updateProjectV2ItemFieldValue` to set `Status` back to the `from` value (or `Refinement` when `from` was empty or invalid) and updates the `<!-- tbhb-dev:issue-checks:v1 -->` comment with a `Board gate` notice naming the blocked target stage and the missing requirement.
4. **Latency and unenforced window.** GitHub Projects v2 offers no synchronous pre-move hook. A direct gateway worker reverts an invalid move in 2 to 5 seconds; a Stage 1 `repository_dispatch` run takes 10 to 30 seconds while a runner starts. Draft cards created directly on a board (`content_type: "DraftIssue"`) have no repository issue and are always reverted to `Backlog`. Because a card can sit in `Ready` during that brief revert window, board `Status` is never trusted alone to start work.

### Command-line behavior in tools

- **`tbhbdev alpha issue create` and `tbhbdev alpha issue edit`:** Both commands render template Markdown when given `--template <variant>` and section values, or read `--body-file`. Both always run the secret scan, the private-detail routing check, and the agent session trailer check first, refusing any violation immediately. Without `--draft`, both run `tbhbdev.content_vet.check` and Vale locally and exit non-zero on any refusal. With `--draft`, schema, kernel-list, and Vale refusals print to standard error as warnings, and the command writes the issue with `check:unchecked` so the workflow evaluates it and posts the findings comment.
- **`tbhbdev alpha agent exec`:** When invoked for a build run (`--profile build` or `--role builder`) that references an issue, the launcher queries the issue's live labels before creating a worktree. It refuses to start if `check:passing` is absent or if `status:unverified` or `status:needs-info` is present. Runs launched with `--role refiner` bypass this stop so refiners can edit and verify failing drafts.

### Cross-role pieces and build order

| Order | Piece | Owner | Repository |
| --- | --- | --- | --- |
| 1 | `tbhb-dev-issue-checks` GitHub App, `issue-checks` environment, five check/status labels, and work board `Status` options (`Backlog`, `Refinement`, `Ready`, `In progress`, `In review`, `Done`) with built-in close and merge workflows turned off | Organization administrator | Organization settings |
| 2 | Tagged CUE variant schemas and templates with required `Out of scope` (`None.` allowed) and prefixed list IDs (`AC1:`, `V1:`, `S1:`) | `pm` | `tbhb-dev/pm` |
| 3 | Headless `content vet-issue` entry point, template rendering, `alpha issue edit`, `--draft` on `create` and `edit`, `check:*` label reservation, and `alpha agent exec` label gate | tools | `tbhb-dev/tools` |
| 4 | Service-management variant alignment for Problems, Requests, and Decisions, plus `Incident` and `Change` stage exemptions | `sm` | Private records repository |
| 5 | Reusable `.github/workflows/issue-checks.yml` workflow, caller workflows in issue-enabled repositories, and one-time backfill run | Rules maintainer and `pm` | `tbhb-dev/rules` and issue repositories |
| 6 | Organization webhook subscription for `projects_v2_item` and gateway route to the board guard | Gateway maintainer | `tbhb-dev/event-gateway` |

## How the 784-item tracker import uses the checks

The staged tracker import holds 784 items across Chores, Defects, Features, Problems, Tasks, Epics, Research, and Spikes. Across those eight types, 247 drafts carry front-matter `status: Unverified` or `status: Needs info`.

1. **Keep the staging gate for text shape and prose.** Under [D5](decisions.md#d5-should-the-tracker-import-keep-the-staging-gate-for-schema-and-prose-checks), the imports role continues to resolve drops and splits in staging and vets every surviving file against its destination CUE schema and the Vale pass before creating an issue ([tracker import](../pm/E-import.md#staging-and-refinement)). For a Defect with `status: Needs info` whose reproduction steps are still unknown, the schema accepts the exact placeholder paragraph `Find a reproduction and add it here.` ([Definition of Ready](../issue-refinement/B-definition-of-ready.md#no-procedure)).
2. **Map front-matter `status:` to the checks label set.** When the importer creates each issue in batches of 25 under the 200-writes-per-hour budget, it strips `status:` from the frontmatter and applies `status:unverified` (for `status: Unverified`) or `status:needs-info` (for `status: Needs info`), as settled in [D7](decisions.md#d7-where-do-the-front-matter-status-values-in-the-imported-drafts-go).
3. **Effect on the board, notifications, and agent work.** Because each created issue already satisfies its CUE variant and Vale rules, the `opened` run of `issue-checks.yml` applies `check:passing` and posts zero failure comments, avoiding 784 comment notifications. Verified items with a passing batch review enter `Ready`. The 247 items carrying `status:unverified` or `status:needs-info` land in `Backlog` or `Refinement` with `check:passing` plus their `status:*` label; both the board guard and `tbhbdev alpha agent exec` hold them out of `Ready` and out of builder runs until a refiner verifies the claim or resolves the question and clears the label.
