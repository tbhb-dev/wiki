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

Today the content kernel refuses to create or edit any issue that misses a required section or fails a Configure Unify Execute (CUE) rule. That all-or-nothing gate blocks rough bug reports and partial drafts when the caller lacks every answer up front.

This design lets callers file draft issues and moves validation into a GitHub Actions workflow that acts like pull request status checks. On issue creation, body edits, label changes, and comments, a dedicated GitHub app validates the issue against the project-management (`pm`) and service-management (`sm`) schemas. It sets one check label (`check:unchecked`, `check:failing`, or `check:passing`) alongside the draft-status labels (`status:unverified` and `status:needs-info`), and updates one comment in place with every finding.

For the operator, incomplete ideas can enter the backlog right away without weakening the readiness gate. Failing or unverified issues can't enter `Ready` on the board and can't start a builder run.

## Proposal

When built, six components enforce the issue checks across public and private repositories where GitHub Issues are on:

- **The checks app** (`tbhb-dev-issue-checks`, owner organization administrator) holds `issues: write`, `projects: write`, and `metadata: read`. Only this app sets the `check:*` labels, updates the single findings comment, and reverts invalid board moves, so an agent account can't mark its own issue valid.
- **The reusable check workflow** (`.github/workflows/issue-checks.yml` in `tbhb-dev/rules`, owner rules maintainer) runs on `issues` and `issue_comment` events. It runs the content kernel's normalizer, `cue vet`, kernel relation checks, Vale prose rules, and secret scans, then writes the check label and updates the findings comment.
- **The checks label set** (owner `pm` and tools) has three mutually exclusive check labels (`check:unchecked`, `check:failing`, and `check:passing`) plus two draft-status labels (`status:unverified` and `status:needs-info`).
- **The work board** (owner `pm`, set up by the organization administrator) uses the built-in GitHub Projects `Status` field with six stage options (`Backlog`, `Refinement`, `Ready`, `In progress`, `In review`, and `Done`). Built-in close and merge workflows stay off so only verified transitions move an item.
- **The board guard** (owner gateway maintainer and `pm`) receives `projects_v2_item` events from the organization webhook through `tbhb-dev/event-gateway`. When a move enters a gated column such as `Ready` or `In progress` without passing checks, it reverts `Status` and notes the unmet requirement in the findings comment.
- **The command-line gates** (`tbhbdev alpha issue` and `tbhbdev alpha agent exec` in `tbhb-dev/tools`, owner tools) render template Markdown, support `create --draft` and `edit --draft`, strip `check:*` from caller labels, and block builder runs on any issue that lacks `check:passing` or carries `status:unverified` or `status:needs-info`.

## How it works

### Validation and the label set

Validation runs the same check pipeline in the command-line tool and in the GitHub Actions workflow. The normalizer parses the title and Markdown body into the block model of sections, lists, and word metrics ([appendix A](A-evidence.md)). Next, `cue vet --all-errors` checks the block model against the shared schemas in `tbhb-dev/tools` and the variant schema from `tbhb-dev/pm` or the private records repository. The runner then checks parent types, sub-issue visibility, the Vale prose rules, and the gitleaks secret and private-detail rules.

| Label | Kind | Meaning | Set and cleared by |
| --- | --- | --- | --- |
| `check:unchecked` | Check state | The current body hasn't finished a validation run yet | Applied on `--draft` creation, replaced by the checks app after evaluation |
| `check:passing` | Check state | The title, body, fields, labels, and parent pass schema, kernel, and Vale checks | Set by the checks app when validation finds zero refusals |
| `check:failing` | Check state | The issue fails one or more schema, kernel, or Vale rules listed in the comment | Set by the checks app when validation finds a refusal |
| `status:unverified` | Draft status | Claims, scope, or reproduction steps haven't been confirmed by a read-only check | Set by the tracker import or intake, cleared by `tbhbdev pm refine` after verification |
| `status:needs-info` | Draft status | An unanswered factual or intent question blocks readiness | Set by the tracker import or a `needs-info` refinement record, cleared after the answer enters the body |

The draft-status labels are independent of the check state labels. An imported issue can pass CUE shape checks (`check:passing`) while still carrying `status:unverified` or `status:needs-info` during the `Refinement` stage. [Appendix C](C-checks-and-board.md#labels-and-loop-prevention) details loop prevention, edit races, and the single comment tagged `<!-- tbhb-dev:issue-checks:v1 -->`.

### Board transitions and enforcement

GitHub lists `projects_v2_item` as an organization webhook event, not a GitHub Actions workflow trigger ([appendix A](A-evidence.md#github-platform-facts)). Because board moves don't start an Actions workflow directly, `tbhb-dev/event-gateway` receives the organization webhook and routes `projects_v2_item` edits to the board guard.

| Target stage | Allowed from | Gate enforced by the board guard | Who moves it |
| --- | --- | --- | --- |
| `Backlog` | New item, `Refinement`, `Ready` | Free for any check or draft-status label | Content kernel, checks app, refiner, or operator |
| `Refinement` | `Backlog`, `Ready`, `In progress` | Free to enter when refinement starts or an edit breaks `check:passing` | Refiner role, checks app on failing edit, builder via `--return`, or operator |
| `Ready` | `Refinement`, `Backlog`, `In progress` | Requires `check:passing`, neither `status:unverified` nor `status:needs-info`, a decided `Out of scope` section (`None.` is valid), and a current `ready` record | Sync job after refinement, or operator |
| `In progress` | `Ready`, `In review` | Meets all `Ready` rules plus an active agent run, assignee, or linked branch | Launcher on run start, sync job, or operator |
| `In review` | `In progress`, `Ready` | Meets all `Ready` rules plus an open non-draft pull request linked to the issue | Sync job on pull request event, or operator |
| `Done` | `In review`, `In progress`, or any stage on `not planned` close | Issue is closed on GitHub, with `check:passing` and verification evidence when closed as completed | Sync job on issue close or pull request merge, or operator |

When an unauthorized move drags a failing or unverified item into `Ready` or `In progress`, the guard calls `updateProjectV2ItemFieldValue` to revert `Status` within a few seconds and updates the findings comment. [Appendix C](C-checks-and-board.md#board-enforcement-and-command-line-gates) covers latency, what stays unenforced on the board, the command-line flags, and how the 784-item tracker import uses the checks.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| Rapid edits cause concurrent runs to flap labels or write a stale comment | The label or comment reflects an older revision of the body | Per-issue Actions concurrency with `cancel-in-progress: true`, plus a final Secure Hash Algorithm (SHA) body digest check before writing | New |
| An agent or script edits an issue in a tight loop | Edit storms burn Actions minutes and hit secondary rate limits | The workflow ignores `tbhb-dev-issue-checks[bot]` events, and the gateway alerts on more than 5 edits to one issue in 10 minutes | Existing for bot sender check, new for alert |
| A web edit in a public issue adds a secret or a private name | A naive findings comment could quote the secret in public | The workflow fails closed, sets `check:failing`, and prints only the rule identifier and line number for scan hits | Existing scan rules, new comment redaction |
| The checks app private key leaks or is misused | A caller could forge `check:passing` on an unverified issue | Key lives in 1Password and a default-branch environment, and the launcher still checks the current refinement record hash | Proven for environment branch policy |
| GitHub Projects allows a card drag before the webhook revert runs | An unready issue briefly shows `Ready` for 2 to 30 seconds | The board guard reverts the field on `projects_v2_item`, and `tbhbdev alpha agent exec` checks the issue's labels directly before starting work | Existing webhook docs, new guard |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | The administrator creates `tbhb-dev-issue-checks`, syncs the five labels, and configures the work board's six `Status` options with built-in close and merge workflows off | A read of the board fields and repository labels matches the tables in [appendix C](C-checks-and-board.md) | Organization administrator |
| 2 | `tbhb-dev/pm` tags the schemas and templates with required `Out of scope` and prefixed list IDs | `bazelisk test //...` passes in `tbhb-dev/pm` and every template passes `cue vet` against its variant | `pm` |
| 3 | `tbhb-dev/tools` adds template rendering, `issue edit`, `--draft` on `create` and `edit`, the headless issue vet command, and the `agent exec` check-label gate | `pytest` in `tbhb-dev/tools` proves `--draft` files an incomplete body with `check:unchecked` while `agent exec` refuses a builder run on a `check:failing` or `status:unverified` issue | tools |
| 4 | `tbhb-dev/rules` ships the reusable `issue-checks.yml` workflow and installs the caller in issue-enabled repositories, plus `sm` wires service variants | Filing a draft issue creates one `<!-- tbhb-dev:issue-checks:v1 -->` comment with findings, and fixing the body flips the label to `check:passing` and updates that comment in place | Rules maintainer, `pm`, and `sm` |
| 5 | `tbhb-dev/event-gateway` routes `projects_v2_item` edits to the board guard, and the backfill run checks existing open issues | Dragging a `check:failing` test issue into `Ready` reverts its `Status` to `Refinement` and updates its findings comment | Gateway maintainer and `pm` |

## Appendices

- [Decisions](decisions.md): 5 open items, then the settled record.
- [A. Evidence](A-evidence.md): what the schemas, tools code, and GitHub documentation show, labeled proven, existing, new, or unverified.
- [B. Alternatives considered](B-alternatives.md): options not taken for labels, board enforcement, draft flags, and the tracker import.
- [C. Checks, board gates, and import](C-checks-and-board.md): the workflow triggers, comment format, command-line behavior, cross-role build order, and the 784-item import.
- A private appendix holds the private repository specifics for this design.
