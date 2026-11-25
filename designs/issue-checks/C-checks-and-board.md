---
type: Design Appendix
title: Checks, board gates, and import
description: Detailed mechanics for the check workflow, the findings comment, the board guard, command-line stops, and the 784-item tracker import.
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

The reusable workflow `.github/workflows/issue-checks.yml` in `tbhb-dev/rules` runs the exact headless check that `tbhbdev` runs locally (`tbhbdev content vet-issue`). Every run does the same two things whatever started it: it validates the live issue, then it runs the [board guard](#how-the-guard-decides).

1. **Read schema pins.** The job loads `schemas/core.cue`, `schemas/issue.cue`, and `schemas/taxonomy.cue` from the pinned `tbhb-dev/tools` wheel, together with `schemas/issue-sections.cue` and `schemas/issue.<variant>.cue` from the pinned `tbhb-dev/pm` commit (or the service variant schema passed by the private records repository).
2. **Read the validated inputs.** The job reads the live title, body, issue type, `area/` and `role/` labels, draft-status labels, organization issue fields, and parent link with the parent's type and visibility. It computes the [input digest](#the-single-findings-comment-and-the-check-record) over them, and `tbhbdev.content_model.normalize` parses the title and body into the JSON block model.
3. **Evaluate CUE.** `cue vet --all-errors` checks section order, required sections, word and character caps, and one-based prefixed list items (`AC1:`, `V1:`, `S1:`). It also checks that `Out of scope` is present (listing exclusions or a single `None.` item, except on the temporary `imported` variant before retirement).
4. **Run kernel and relation checks.** Using the checks app token, the runner verifies parent type rules (`Initiative` for an `Epic`, `Epic` for a `Feature`, non-`Task` for a `Task`, `Problem` or `Epic` for a `Defect` with a parent) and public-to-private visibility rules. It also checks for links-only sections, note flooding, title shape warnings, and a [draft-status label that disagrees with its record](#who-writes-each-label).
5. **Run Vale and scans.** The runner runs `vale --ext=.md` with the organization's `.vale.ini` and styles from `tbhb-dev/rules`, applying the short-body rule that downgrades errors to warnings when prose is under half the variant's word cap. Finally, it runs the gitleaks secret scan and private-detail scan on the title and body.
6. **Write the result.** The runner reads the validated inputs again and exits without writing when their digest changed during the run, because a newer run covers the newer inputs. Otherwise it writes the check label and the check record.

Each run executes on a standard Linux runner with zero large language model (LLM) calls and zero paid external services. It makes 2 to 4 GitHub API requests and finishes in 8 to 15 seconds.

## Labels and loop prevention

### Who writes each label

Each label has one writer ([D13](decisions.md#d13-who-writes-each-label-and-what-happens-on-an-unauthorized-label-change)), and the [consistency table](README.md#consistency-table) lists the gates that read it.

- **The checks app writes all three `check:*` labels.** It applies `check:unchecked` as the first write of the first run on an issue that has no check label, then replaces it with `check:passing` or `check:failing` when the run completes, except for the one deferred case in [the import section](#how-the-784-item-tracker-import-uses-the-checks). The command-line tool, the tracker importer, and the refiner never write a `check:*` label, and the draft-filing path files an issue with no check label at all.
- **The content kernel writes both draft-status labels.** Its import verb sets `status:unverified` or `status:needs-info` when it files a draft ([D7](decisions.md#d7-where-do-the-front-matter-status-values-in-the-imported-drafts-go)). Its refine verb (`tbhbdev pm refine`) sets `status:needs-info` on a `needs-info` outcome and clears either label after verification or after the answer enters the body.
- **The authoritative source for the draft-status labels is the issue's current kernel record.** That record is the latest refinement record on the issue or, for an imported issue with no later refinement record, the staging record on its batch Chore, which also records the label the importer mapped. The kernel writes the record first and the label last, and an issue with no kernel record carries neither label.

GitHub has no per-label access control list (ACL), so any account with `issues: write` could call the label API directly ([appendix A](A-evidence.md#github-platform-facts)). Three controls keep the single-writer rule:

1. **Command-line refusal.** `tbhbdev.content` and `tbhbdev alpha issue` reject any `--label` argument that starts with `check:`, and accept `status:unverified` or `status:needs-info` only from the import and refine verbs. Agent harness rules deny raw `gh issue edit` label commands.
2. **A changed `check:*` label is rewritten.** A `labeled` or `unlabeled` event on a `check:*` label from any sender other than the checks app starts a run, and that run writes the label its own result gives.
3. **A changed draft-status label is compared with its record.** The event sender can't tell the kernel from another session on the same account, so the run compares the live draft-status labels with the current kernel record instead. An importer or refiner change matches its record and stands, while a manual change disagrees with the record, so the run reports the refusal `draft-status-mismatch` and sets `check:failing` until the label matches the record again.

The checks app never writes a draft-status label. A person who restores the label named in the refusal starts a run that clears the finding.

### The follow-up check after filing

The kernel opens an issue first, then writes its organization fields, attaches its parent, and adds it to the project. The `opened` run can therefore read an issue whose filing writes aren't finished. The kernel filing path (`issue create`, `issue create --draft`, and the tracker importer) applies the issue's labels as its last filing write, after those writes and after the importer's record on the batch Chore ([D11](decisions.md#d11-how-does-an-issue-get-checked-again-after-the-kernels-last-filing-write)).

Those `labeled` events come from the kernel's account and match the label filter, so they start a run that reads the finished issue. That run cancels the `opened` run under the per-issue concurrency group, or replaces its result within seconds. An issue the kernel files with no label has no such event, so the scheduled sweep checks it within one sweep interval.

### Loop prevention and edit races

- **Triggers.** The workflow runs on `issues` and `issue_comment` events, on a `repository_dispatch` event that the gateway sends for a board move, and on a schedule every 15 to 30 minutes. The scheduled sweep validates each open issue whose check record is missing or whose input digest differs from the live inputs, and runs the board guard for every card.
- **Sender guard.** The workflow's job-level `if:` condition skips any `issues` or `issue_comment` event where `github.event.sender.login == 'tbhb-dev-issue-checks[bot]'`. A label swap or comment update by the checks app never starts a second run.
- **Label filter.** For other senders on `labeled` and `unlabeled`, the workflow runs only when `github.event.label.name` starts with `check:`, `status:`, `area/`, or `role/`.
- **Concurrency and input digest.** Each run joins one concurrency group keyed by repository and issue number, with `cancel-in-progress: true`. A canceled run loses nothing, because the newer run validates the live issue and runs the guard again. Step 6 above stops a run from writing a result for inputs that changed while it ran.
- **Edit-storm protection.** If a single issue triggers more than 5 check runs within 10 minutes, the workflow logs a rate warning and `tbhb-dev/event-gateway` emits an alert so the operator or coordinator can stop a runaway loop.

### The single findings comment and the check record

Each issue has at most one findings comment. The checks app creates it once, on the first completed run, and every later run and every board guard message updates that same comment in place. Nothing in this design creates a second comment on an issue.

The app finds the comment by its author (`user.login == 'tbhb-dev-issue-checks[bot]'`) together with the hidden marker `<!-- tbhb-dev:issue-checks:v1 -->`. The lookup never uses the webhook event's `sender.login`, because the sender of an issue edit is the editor, never the app. The per-issue concurrency group keeps two runs from both creating the comment.

The comment starts with the check record, which is the one freshness signal for a check result. The record holds the result and `inputs_sha256`, the input digest: a Secure Hash Algorithm (SHA) 256-bit digest over a canonical form of every validated input from step 2 above. A title-only edit, a parent change, or a draft-status label change each give a different digest, and no reader compares a body-only hash, a timestamp, or the comment text.

```markdown
<!-- tbhb-dev:issue-checks:v1 -->
<!-- tbhb-dev:check-record result=failing inputs_sha256=9f4c... -->
### Issue checks: failing

Evaluated variant `feature` at inputs `sha256:9f4c...` on 2026-11-25T14:20:00Z. Fix the refusals below by editing the issue.

| Kind | Section | Rule | Limit | Actual | Hint |
| --- | --- | --- | --- | --- | --- |
| Refusal | Acceptance criteria | prefixed-list-id | `^AC1: \S` | `- First check.` | Start each bullet with `AC1: ` and count up by one. |
| Refusal | Out of scope | required-section | present | missing | List deliberate exclusions, or write `- None.` after considering scope. |
| Warning | Title | title-shape | `<Role> can <action>` | `Add check labels` | Use a role and what it can do. |

### Board gate

The card left `Ready` for `Refinement` on 2026-11-25T14:21:00Z because the issue lacks `check:passing`.
```

A passing run writes `result=passing` with the digest of the inputs it validated, under the heading `### Issue checks: passing`, with no findings table. The `Board gate` section belongs to the board guard, which edits only that section. A rejected move never changes the check record, the check result, or the `check:*` label, so a passing issue stays passing after the guard moves its card.

If a web edit in a public repository trips a secret or private-detail scan rule, the table prints only the rule ID and line number and never quotes the offending text.

### Existing issues

Open issues created before the workflow ships carry no `check:*` label and no check record. A `workflow_dispatch` backfill mode on `issue-checks.yml` runs in rollout step 4 under the 200-writes-per-hour budget, sets `check:passing` or `check:failing` on each open issue, and writes its findings comment. The launcher gate ships last, in rollout step 6, so every valid existing issue has a passing record before any builder run depends on one.

## Board enforcement and command-line gates

### One writer for the board status field

The board guard is the last job of `issue-checks.yml` and runs with the checks app credential. It's the only automated writer of the built-in `Status` field on the work board ([D12](decisions.md#d12-which-component-writes-the-board-status-field-and-how-does-it-avoid-stale-writes)). It advances a refined or imported issue into `Ready` and the later stages, moves back a card that a person dragged into a stage its issue doesn't qualify for, and demotes a card whose issue stops qualifying.

The sync job keeps the one field it has today. It alone writes the organization issue field under [status transitions](../issue-refinement/D-status-transitions.md#transition-order), and it never writes the board `Status` field.

### How the guard decides

The guard runs at the end of every workflow run, so it has three triggers:

- **A check run.** An issue or comment event, such as an edit, a label change, a new refinement record, or a close, starts a run that validates and then runs the guard.
- **A board move.** GitHub sends a `projects_v2_item` (`action: edited`) webhook to `tbhb-dev/event-gateway`. When the changed field is `Status` and `sender.login` isn't `tbhb-dev-issue-checks[bot]`, the gateway sends a `repository_dispatch` event to `issue-checks.yml` in the issue's repository, whatever column the card entered.
- **The scheduled sweep.** It catches a change that raises no issue event, such as a linked pull request opening or a parent changing type, and any missed delivery.

The event only starts the guard. The guard never acts on the `from` and `to` values in a payload, because they describe a past state.

1. **Read live state.** Immediately before any write, the guard reads the card's live `Status`, the issue's live labels, state, linked pull requests, and `Out of scope` section, and its readiness record.
2. **Find the readiness record.** The guard accepts either a current `ready` refinement record on the issue or an approved staging record on a tracker-import batch Chore, as [current records](../issue-refinement/D-status-transitions.md#current-records) defines them. It finds the staging record through the `cross-referenced` event that the Chore's recorded issue link leaves on the issue's timeline (unverified). It then confirms that the Chore sits under the import Epic with a passing batch-review verdict, and that the recorded issue link and vetted body hash match the live issue.
3. **Compute the qualifying stage.** This is the latest gated stage in [the README stage table](README.md#board-stages-and-the-guard) whose gate holds for the live issue, or none.
4. **Write only on a difference.** When the issue has a qualifying stage and the card sits elsewhere, the guard calls `updateProjectV2ItemFieldValue` to put the card in that stage. When no stage qualifies, a card in `Backlog` or `Refinement` stays where it is, and a card in a gated column moves to `Refinement`.
5. **Explain a rejected placement.** When the guard moves a card out of a gated column, it updates the `Board gate` section of the findings comment with the column the card left and the unmet requirement. If the issue has no findings comment yet, the guard creates the one comment with the marker and a record of `result=unchecked`.

Because every write follows from state read at write time, a delayed guard run can't overwrite a later valid move or restore a stage that became invalid. The same rule covers a card that sits in `Ready` when a routine edit makes the issue fail: the edit's own run sets `check:failing`, and the guard then moves the card to `Refinement`. A `DraftIssue` card has no repository issue to check or comment on, so the guard returns it to `Backlog` from any gated column.

### Revert latency and what stays unenforced

GitHub Projects v2 offers no synchronous pre-move hook (unverified), so the guard can't block a drag. An invalid drag into `Ready` shows there until the guard moves the card back after the fact, in an estimated 10 to 30 seconds through `repository_dispatch` (unverified, measured in plan step 5). While the card sits in `Ready` during that window, the check labels, the draft-status labels, and the launcher gate still hold.

The guard enforces every `Status` move, including a move into `Backlog` or `Refinement`. A person who drags a qualifying card back to `Backlog` sees the guard advance it again, so the way to hold an item back is `tbhbdev pm refine --return`, which makes its record stale. These things stay unenforced:

- The choice between `Backlog` and `Refinement` for an issue with no qualifying stage.
- The identity of whoever moves a card.
- Card order within a column.
- Every project field other than `Status`.

The guard's read and its write aren't one atomic call, so a move that lands in the gap between them (unverified, under a second) can be overwritten. That move's own event starts another run, which decides again from live state.

### Command-line behavior in tools

- **`tbhbdev alpha issue create` and `tbhbdev alpha issue edit`:** Both commands render template Markdown when given `--template <variant>` and section values, or read `--body-file`. Both always run the secret scan, the private-detail routing check, and the agent session trailer check first, and refuse any violation immediately. Without `--draft`, both run `tbhbdev.content_vet.check` and Vale locally and exit non-zero on any refusal.
- **The draft-filing path:** With `--draft`, schema, kernel-list, and Vale refusals print to standard error as warnings and the command files the issue through the same kernel write path. It writes no `check:*` label, and the checks app sets `check:unchecked` on the first run and then the result.
- **`tbhbdev alpha agent exec`:** For a build run (`--profile build` or `--role builder`) that references an issue, the launcher reads the live issue before it creates a worktree. It refuses to start unless the issue has `check:passing`, has neither `status:unverified` nor `status:needs-info`, and has a check record of `result=passing` whose `inputs_sha256` equals the digest the launcher computes from the live inputs. The launcher computes that digest with the same kernel function as the validator.
- **Trusting the record:** The launcher reads the record only from the comment whose author and last editor are both `tbhb-dev-issue-checks[bot]` (unverified). A missing record, a second marker comment, or any digest mismatch is a refusal.
- **Refiner runs:** Runs launched with `--role refiner` skip the launcher stop so refiners can edit and verify failing drafts.

### Cross-role pieces and build order

| Order | Piece | Owner | Repository |
| --- | --- | --- | --- |
| 1 | `tbhb-dev-issue-checks` GitHub App, `issue-checks` environment, the five labels, and work board `Status` options (`Backlog`, `Refinement`, `Ready`, `In progress`, `In review`, `Done`) with built-in close and merge workflows turned off | Organization administrator | Organization settings |
| 2 | Tagged CUE variant schemas and templates with required `Out of scope` (`None.` allowed) and prefixed list IDs (`AC1:`, `V1:`, `S1:`) | `pm` | `tbhb-dev/pm` |
| 3 | Headless `content vet-issue` entry point with the input digest, template rendering, `alpha issue edit`, `--draft` on `create` and `edit`, refusal of caller `check:*` labels, and labels as the last filing write | tools | `tbhb-dev/tools` |
| 4 | Service-management variant alignment for Problems, Requests, and Decisions, plus `Incident` and `Change` stage exemptions | `sm` | Private records repository |
| 5 | Reusable `.github/workflows/issue-checks.yml` workflow with the validator and the scheduled sweep, caller workflows in issue-enabled repositories, and the one-time backfill across existing open issues | Rules maintainer and `pm` | `tbhb-dev/rules` and issue repositories |
| 6 | Organization webhook subscription for `projects_v2_item`, the gateway route, and the board guard job | Gateway maintainer and rules maintainer | `tbhb-dev/event-gateway` and `tbhb-dev/rules` |
| 7 | The fail-closed launcher gate in `alpha agent exec`, enabled only after pieces 5 and 6 pass their checks | tools | `tbhb-dev/tools` |

## How the 784-item tracker import uses the checks

The staged tracker import holds 784 items across Chores, Defects, Features, Problems, Tasks, Epics, Research, and Spikes. Across those eight types, 247 drafts carry front-matter `status: Unverified` or `status: Needs info`.

1. **The staging gate for text shape and prose.** [pm decision D11](../pm/decisions.md#d11-which-issue-types-does-the-tracker-import-create) already has the imports role resolve drops and splits in staging and vet every surviving file against its destination CUE schema and the Vale pass before the kernel creates it ([tracker import](../pm/E-import.md#staging-and-refinement)). [D5](decisions.md#d5-should-the-tracker-import-keep-the-staging-gate-for-schema-and-prose-checks) asks whether to keep that gate now that drafts can be filed.
2. **Front-matter `status:` maps to the draft-status labels.** When the kernel's import verb files each issue in batches of 25 under the 200-writes-per-hour budget, it strips `status:` from the front matter and applies `status:unverified` (for `status: Unverified`) or `status:needs-info` (for `status: Needs info`). [D7](decisions.md#d7-where-do-the-front-matter-status-values-in-the-imported-drafts-go) settles that mapping, and it holds under either answer to D5.
3. **A Defect draft with no reproduction.** The [Definition of Ready](../issue-refinement/B-definition-of-ready.md#no-procedure) allows the placeholder `Find a reproduction and add it here.` on a `needs-info` item, but the Defect variant rejects it (unverified), and this design proposes no `pm` schema change. The importer files such a Defect through the draft-filing path with `status:needs-info`. While a Defect carries that label and the placeholder, a check run performs only the scans and records the outcome `check:unchecked`.
4. **Validation after refinement.** When a refiner replaces the placeholder with a reproduction, that edit starts a full check run against the Defect variant. The import counts the item as validated only when that run sets `check:passing`.
5. **Effect on the board, notifications, and agent work.** Each issue that passed staging gets `check:passing` and one findings comment with a passing record and no findings table. The board guard advances verified items with a passing batch review into `Ready`. The 247 items with a draft-status label stay in `Backlog` or `Refinement`, and the launcher refuses builder runs on them until a refiner verifies the claim or resolves the question and clears the label.
