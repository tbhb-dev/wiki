---
type: Design Draft
title: Status checks for issues
description: How a validation workflow, check labels, and board gates let people and agents file draft issues while stopping unready work.
tags: [issues, checks, projects, pm, design]
status: draft
owner: pm
updated: "2026-11-25"
decisions_open: 5
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Status checks for issues

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | pm | 2026-11-25 | 5 open choices in [decisions.md](decisions.md), about 5 minutes |

## Summary

Today the content kernel refuses to create or edit any issue that misses a required section or fails a Configure Unify Execute (CUE) rule. That all-or-nothing gate blocks rough bug reports and partial drafts when details are still unknown.

This design lets callers file draft issues and moves validation into a GitHub Actions workflow that acts like pull request status checks. A dedicated GitHub app validates each issue against the project-management (`pm`) and service-management (`sm`) schemas. It alone writes one check label (`check:unchecked`, `check:failing`, or `check:passing`) and one findings comment that it updates in place.

A board guard keeps each card in the stage its issue qualifies for. It can't block a drag into `Ready`, so it moves the card back after the fact, within an estimated 10 to 30 seconds (unverified). During that window the check labels and the launcher gate still stop builder runs.

## Proposal

When built, six components enforce the issue checks across public and private repositories where GitHub Issues are on:

- **The checks app** (`tbhb-dev-issue-checks`, owner organization administrator) holds `issues: write`, `projects: write`, and `metadata: read`. It's the only writer of the three `check:*` labels and the findings comment, and the only automated writer of the board `Status` field, so an agent account can't mark its own issue valid.
- **The reusable check workflow** (`.github/workflows/issue-checks.yml` in `tbhb-dev/rules`, owner rules maintainer) runs on `issues` and `issue_comment` events, on a board-move dispatch from the gateway, and on a schedule. Every run validates the live issue with the content kernel's normalizer, `cue vet`, kernel relation checks, Vale prose rules, and secret scans, and then runs the board guard.
- **The label set** (owner `pm` and tools) has three mutually exclusive check labels (`check:unchecked`, `check:failing`, and `check:passing`) plus two draft-status labels (`status:unverified` and `status:needs-info`). Each label has one writer, as the [consistency table](#consistency-table) shows.
- **The work board** (owner `pm`, set up by the organization administrator) uses the built-in GitHub Projects `Status` field with six stage options (`Backlog`, `Refinement`, `Ready`, `In progress`, `In review`, and `Done`). Built-in close and merge workflows stay off so only the board guard moves an item automatically.
- **The board guard** (owner rules maintainer and gateway maintainer) is the last job of the check workflow. It reads the card's live column and the issue's live state, then advances, moves back, or demotes the card so it sits in the stage the issue qualifies for.
- **The command-line gates** (`tbhbdev alpha issue` and `tbhbdev alpha agent exec` in `tbhb-dev/tools`, owner tools) render template Markdown, support `create --draft` and `edit --draft`, and never write a `check:*` label. The launcher refuses a builder run unless the issue has `check:passing`, no draft-status label, and a passing check record that covers the live issue.

## How it works

### Validation and the check record

Validation runs the same check pipeline in the command-line tool and in the GitHub Actions workflow. The normalizer parses the title and Markdown body into the block model of sections, lists, and word metrics ([appendix A](A-evidence.md)). Then `cue vet --all-errors` checks the block model against the shared schemas in `tbhb-dev/tools` and the variant schema from `tbhb-dev/pm` or the private records repository, and the runner checks relations, Vale prose rules, and the secret and private-detail rules.

Each issue has one findings comment, which the checks app creates once and updates in place. The app finds it by its own bot login as the comment's author plus the hidden marker `<!-- tbhb-dev:issue-checks:v1 -->`, never by the sender of an event. Board guard messages go in a separate `Board gate` section of that same comment, and a rejected move never changes the check result.

The comment starts with the check record, which holds the result and one input digest (`inputs_sha256`). The digest covers the title, the body, and every other input the validator reads, so a title-only edit makes a passing record stale. The launcher computes the same digest from the live issue and refuses a builder run on any mismatch ([appendix C](C-checks-and-board.md#the-single-findings-comment-and-the-check-record)).

### Consistency table

| Label | Single writer | When the writer sets it | Transitions and gates that read it |
| --- | --- | --- | --- |
| `check:unchecked` | Checks app | As the first write of the first run on an issue with no check label, and as the outcome for an imported Defect draft that still lacks a reproduction. The command-line draft path and the importer never write it | The board guard keeps the card out of `Ready`, `In progress`, `In review`, and a completed `Done`. The launcher refuses builder runs |
| `check:failing` | Checks app | When a completed run finds a refusal, including a draft-status label that disagrees with its record | The board guard keeps the card out of the same stages and demotes a card that sits in one, such as a `Ready` card after a routine edit. The launcher refuses builder runs |
| `check:passing` | Checks app | When a completed run finds zero refusals, together with the passing check record and its `inputs_sha256` in the one findings comment | The board guard requires it, read live before every write. The launcher requires it and requires the record's digest to equal the digest of the live title, body, and other validated inputs |
| `status:unverified` | Content kernel | The import verb sets it at filing from front-matter `status: Unverified`. The refine verb clears it after verification | The board guard keeps the card out of `Ready` and every later stage. The launcher refuses builder runs |
| `status:needs-info` | Content kernel | The import verb sets it at filing from front-matter `status: Needs info`, and the refine verb sets it on a `needs-info` outcome. The refine verb clears it once the answer is in the body | The board guard keeps the card out of `Ready` and every later stage. The launcher refuses builder runs. The validator defers the Defect schema while a Defect carries it with the placeholder reproduction |

The draft-status labels follow the issue's current kernel record, which is the latest refinement record or the staging record on the import's batch Chore. The board `Status` field isn't a label, and its only automated writer is the board guard. [Appendix C](C-checks-and-board.md#labels-and-loop-prevention) details the label controls, the follow-up check after filing, and loop prevention.

### Board stages and the guard

GitHub lists `projects_v2_item` as an organization webhook event, not a GitHub Actions workflow trigger ([appendix A](A-evidence.md#github-platform-facts)). So `tbhb-dev/event-gateway` receives the organization webhook and dispatches each `Status` edit to the check workflow, where the board guard runs.

| Stage | Gate the board guard checks against the live issue | Who places a card there |
| --- | --- | --- |
| `Backlog` | None | Any project writer |
| `Refinement` | None | Any project writer, and the board guard when it demotes a card |
| `Ready` | `check:passing`, neither `status:unverified` nor `status:needs-info`, a decided `Out of scope` section (`None.` is valid), and a current `ready` refinement record on the issue or an approved staging record on its import batch Chore | The board guard |
| `In progress` | Every `Ready` rule plus an assignee or a linked branch | The board guard |
| `In review` | Every `Ready` rule plus an open non-draft pull request linked to the issue | The board guard |
| `Done` | The issue is closed on GitHub, with `check:passing` and verification evidence when closed as completed | The board guard |

The guard runs after every check run, on every board-move dispatch, and on a scheduled sweep. An event only starts it. Before any write it reads the card's live column and the issue's live state, so a delayed run never overwrites a later valid move and never restores a stage that became invalid.

The guard is the single writer that advances a refined or imported issue into `Ready` and the later stages. The sync job keeps writing only the organization issue field, as [status transitions](../issue-refinement/D-status-transitions.md#transition-order) describes. When a routine edit makes an issue in `Ready` fail, the run that sets `check:failing` also moves the card to `Refinement`.

GitHub Projects v2 offers no synchronous pre-move hook (unverified), so a person can still drag a card into a gated column. The card shows there until the guard moves it back after the fact, within an estimated 10 to 30 seconds (unverified). [Appendix C](C-checks-and-board.md#board-enforcement-and-command-line-gates) covers the guard's rule, what stays unenforced on the board, the command-line flags, and how the 784-item tracker import uses the checks.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| Rapid edits cause concurrent runs to flap labels or write a stale comment | The label or comment reflects an older revision of the issue | Per-issue Actions concurrency with `cancel-in-progress: true`, plus a second read of the input digest before any write | New |
| The kernel writes fields and the parent link after it opens an issue | The first run validates an unfinished issue and marks a valid issue failing | The kernel applies labels as its last filing write, which starts a follow-up run, and the scheduled sweep covers an issue filed with no label | New |
| An agent or script edits an issue in a tight loop | Edit storms burn Actions minutes and hit secondary rate limits | The workflow ignores `tbhb-dev-issue-checks[bot]` events, and the gateway alerts on more than 5 edits to one issue in 10 minutes | Existing for bot sender check, new for alert |
| A web edit in a public issue adds a secret or a private name | A naive findings comment could quote the secret in public | The workflow fails closed, sets `check:failing`, and prints only the rule identifier and line number for scan hits | Existing scan rules, new comment redaction |
| The checks app private key leaks or is misused | A caller could forge `check:passing` and a passing record | The key lives in 1Password and a default-branch environment, revoking it ends the forgery, and `Ready` still needs a readiness record from the kernel's account | Proven for environment branch policy |
| An account with write access edits the findings comment | A forged record could claim a passing result | The launcher trusts the record only when the comment's author and last editor are both the checks app | Unverified, checked in plan step 4 |
| GitHub Projects v2 has no pre-move hook, so a card drag lands before the guard runs | An unready issue shows `Ready` for an estimated 10 to 30 seconds | The board guard moves the card back after the fact, and the launcher checks the live labels and the input digest before it starts work | Unverified pre-move absence and latency, existing webhook docs, new guard |
| The guard's read and write aren't atomic | A move in the gap of under a second is overwritten | That move's own event starts another guard run, which decides again from live state | Unverified, measured in plan step 5 |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | The administrator creates `tbhb-dev-issue-checks`, syncs the five labels, and configures the work board's six `Status` options with built-in close and merge workflows off | A read of the board fields and repository labels matches the tables in [appendix C](C-checks-and-board.md) | Organization administrator |
| 2 | `tbhb-dev/pm` tags the schemas and templates with required `Out of scope` and prefixed list IDs | `bazelisk test //...` passes in `tbhb-dev/pm`, every template passes `cue vet` against its variant, and a `cue vet` run on a Defect body with the placeholder reproduction records whether the variant rejects it | `pm` |
| 3 | `tbhb-dev/tools` adds template rendering, `issue edit`, `--draft` on `create` and `edit`, refusal of caller `check:*` labels, labels as the last filing write, and the headless `content vet-issue` command with the input digest | `pytest` in `tbhb-dev/tools` proves `--draft` files an incomplete body with no `check:*` label and applies labels last, and `content vet-issue` reports findings and the digest without an agent session | tools |
| 4 | `tbhb-dev/rules` ships the reusable `issue-checks.yml` validator and sweep and installs the caller in issue-enabled repositories, `sm` wires service variants, and the backfill run checks existing open issues | A filed draft gets `check:unchecked`, then `check:failing` and exactly one findings comment, and the API reports the app as its author and last editor. Fixing the body flips the label and the comment in place, a title-only edit changes the recorded `inputs_sha256`, and a run finds a calibration issue's staging record from its timeline. Every existing open issue has a check label and a record | Rules maintainer, `pm`, and `sm` |
| 5 | `tbhb-dev/event-gateway` dispatches `projects_v2_item` `Status` edits, and `tbhb-dev/rules` adds the board guard job | A `check:failing` test issue dragged into `Ready` returns to `Refinement` with a `Board gate` section and an unchanged check record, and the measured latency is on record. A test issue in `Ready` leaves it after an edit that fails, and a test issue with a `ready` record enters `Ready` with no person moving it | Gateway maintainer and rules maintainer |
| 6 | `tbhb-dev/tools` enables the fail-closed launcher gate, last, after the step 4 backfill and step 5 | `agent exec` starts a builder run on a passing test issue. It refuses one when `check:passing` is absent, a draft-status label is present, or the record's digest differs from the live issue after a title-only edit | tools |

## Appendices

- [Decisions](decisions.md): 5 open items, then the settled record.
- [A. Evidence](A-evidence.md): what the schemas, tools code, and GitHub documentation show, labeled proven, existing, new, or unverified.
- [B. Alternatives considered](B-alternatives.md): options not taken for labels, board enforcement, draft flags, and the tracker import.
- [C. Checks, board gates, and import](C-checks-and-board.md): the workflow triggers, comment format, guard rule, command-line behavior, cross-role build order, and the 784-item import.
- A private appendix holds the private repository specifics for this design.
