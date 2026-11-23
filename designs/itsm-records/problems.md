---
type: Design Appendix
title: Service-management records problems
description: How the Problem issue type works, how it relates to Incidents, Defects, and Chores, and how the problems tracker and the staged import become Problems.
tags: [service-management, problems, records, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Service-management records problems

Back to [the design](README.md).

The design below is a draft. The parts that the evidence table cites as merged exist as code, and the rest remains a plan. The private records repository stores each Problem, and each fix goes in the repository with its code.

## The types

An Incident records one interruption and closes when service returns, while a Problem records the cause behind one or more Incidents, behind a recurring failure, or behind a failure nobody can explain.

A Problem has two endings. It ends Resolved after its fixes merge and both its Resolution criteria and its recurrence check pass, or Accepted after a recorded decision to live with the failure.

A Defect is one code fix, and a Chore is one configuration or runbook fix. A code Defect with a clear fix and no recurrence skips the Problem step.

## Non-goals

- The design relies on no close roll-up from GitHub and adds none.
- A known error is a Problem state, never a separate record.
- pm's Status field and its sync job stay as pm defines them, and this design gives the sync job no new write.
- The Problem workflow never runs code from an issue.
- The importer never opens past Incidents.

## Parent or dependency

**Rule:** a Problem is the sub-issue parent of each Defect and Chore that fixes its cause. Any fix that already has a parent, such as an Epic or a second Problem, joins as a "blocked by" dependency instead, because an issue takes one parent. Together, the sub-issues and the "blocked by" dependencies make up the Problem's tracked fixes, and every rule below that names tracked fixes means both kinds.

An Incident cites its Problem in a body line, and the Problem lists the Incident under Related incidents. Both ends link by text alone, which keeps the Incident free of a sub-issue link or a dependency.

**Rejected:** a "blocked by" dependency as the default link to each fix, because it lacks a progress count and pm requires a parent on every Chore. Also rejected: an Incident as a sub-issue or a dependency of its Problem. The Incident closes on restore, well before the Problem, and GitHub would mark it Blocked.

**Tradeoff:** GitHub allows 100 sub-issues per parent and eight levels, across repositories of the same owner. GitHub documents no close roll-up in either direction, so the Problem workflow below moves the Problem when its tracked fixes close.

## Fields

| Fact | Home | Values | Set by |
| --- | --- | --- | --- |
| Type | Native issue type | Problem | The open verb |
| Problem status | Org issue field | Investigating, Known error, Verifying, Resolved, Accepted | See transitions |
| Source | Body metadata | Incident, Recurrence, Check, Review, Import | The open verb, once |
| Category | Body metadata | The existing category list | The open verb |
| Owner | `role:<name>` label | One agent role from the list sm keeps | Refinement |
| Priority | pm's Project field | Expedite, Standard, Intangible | Refinement weighs impact and recurrence |
| Area | `area:<repo>` label | The affected repository | The open verb |

Problem status stays apart from pm's Status, which only pm's sync job writes from GitHub state. A state label would let two states coexist, and body metadata would make each move edit the body and rerun the content check.

The owner is a `role:<name>` label rather than an assignee. Every agent acts as one GitHub user, so an assignee would point at that one user for every role, and a label routes work to a role without proving who made a move.

The design checks the condition of each move instead of the mover. "The owner moves it" works as a routing rule, and the entry rules below carry the enforcement.

## Actors

- **The open verb** creates the Problem through the content kernel, which holds the write token.
- **The owner** is any agent run launched for the Problem's role. It edits the Problem through the same verbs and kernel.
- **The Problem workflow** (planned) runs in the private records repository on Problem issue events and on an hourly schedule, because a fix closing in another repository fires no event there. It takes its token from the records app that the repository already uses, and it writes only Problem status and the open or closed state.
- **The recurrence check** (planned) is a scheduled job in the home repository of the check that a Problem names. It posts one result comment per run on the Problem through the content kernel and leaves every field unchanged.

The Problem workflow needs read access to issues in each repository with a tracked fix. Whether the records app has that access today is unverified.

## Transitions

Each state has an entry rule. A verb refuses a move that breaks it, and the Problem workflow reverts any move it finds that breaks it, with a comment that states the unmet rule.

| State | Entry rule |
| --- | --- |
| Investigating | Source set. Cause may be empty or list candidate causes, numbered |
| Known error | Cause states one confirmed root cause, and Workaround describes a workaround or says None |
| Verifying | Meets the Known error rule, every tracked fix has closed as completed, Resolution criteria list at least one item, and Recurrence check names a check and a window |
| Resolved | Closed as completed. The owner has checked every Resolution criterion with a link to its evidence, and the recurrence check passed every run in the window |
| Accepted | Closed as not planned, with a decision line and a review date, or a note that the component retired. Each tracked fix has closed |

| From | To | Who moves it | Condition |
| --- | --- | --- | --- |
| (new) | Investigating | The open verb or the importer | Source set. This is the only entry state |
| Investigating | Known error | The owner | Meets the Known error rule |
| Known error | Verifying | The Problem workflow | Meets the Verifying rule, and every recurrence run has passed since the latest tracked fix closed, or since the Problem opened when it has zero tracked fixes |
| Verifying | Resolved | The Problem workflow, closing as completed | Meets the Resolved rule, counting only runs after the Problem entered Verifying |
| Verifying | Known error | The Problem workflow, or the owner | A recurrence run failed, or the owner found a Resolution criterion unmet. The window count resets to zero |
| Known error or Verifying | Investigating | The owner | Evidence shows the confirmed cause is wrong |
| Investigating, Known error, or Verifying | Accepted | The owner, closing as not planned | Meets the Accepted rule |
| Resolved or Accepted | Investigating | Any role reopens the issue, and the Problem workflow sets the status | A recurrence after close, or an Accepted review that finds a fix now justifies its cost |

Resolved comes from Verifying alone, through the Problem workflow. If a role closes a Problem as completed first, the workflow reopens it and restores the earlier status.

After a failed verification, the Problem waits in Known error until the owner adds or reopens a fix and that fix closes. That close satisfies the move to Verifying again, and the window starts over.

A tracked fix closed as not planned also blocks Verifying. In that case the owner replaces that fix or closes the Problem as Accepted.

Any open state can end in Accepted when the owner closes the Problem as not planned, after closing each open tracked fix as not planned or removing its link. An agent role may do so on its own decision line. A workaround that loosens a guard or a trust boundary needs an approval Request first.

Open Problems in Known error and closed Problems marked Accepted form the known-error database. An Incident responder searches them for a Workaround.

In the private records repository, the Problem template changes in four places. Suspected cause becomes Cause, which states either the root cause or a numbered list of candidate causes. The template also gains a Workaround section, a Resolution criteria section (a list of the shared `#Criterion` items), and a Recurrence check section (one command or query, its expected result, its window, and a last-run line).

Summary, Related incidents, and Next step keep their names. By default the window lasts 14 days, and a Problem whose check runs daily may set a shorter one.

## Refinement

The wiki lacks a pm refinement design at this head, so this section defines only the Problem's ready check. It follows pm's hierarchy in [the pm design](../pm/README.md): a Defect or Chore links to its parent as a sub-issue.

A Problem is ready for investigation when it meets four conditions:

- **Scope:** one sentence that states the symptoms and the hosts or repositories in scope.
- **Evidence:** at least one dated Incident, occurrence, or failing check.
- **Owner:** one `role:<name>` label.
- **Next step:** one action with a time box. A step too large for that line becomes a Spike sub-issue.

Refinement of a staged Defect has three outcomes. It stays a Defect, it becomes the child of an existing Problem, or it becomes a new Problem. Items that merely look alike link as Related until evidence shows one cause.

Refinement also writes the Cause of an imported Problem. Once the owner confirms one root cause and fills Workaround, the owner moves the Problem to Known error, as each split below shows.

## Worked splits

Each split starts from the staged import candidate list and shows the Problem after refinement, beginning in Investigating with Source Import.

Rows and repositories appear by role, without identifiers.

**P1: a cluster with one cause and a workaround.** A staged item in the tools repository becomes the Problem "Launcher builds drift across hosts and accounts." Refinement confirms the cause: builds go onto each host by hand, so hosts and accounts drift apart.

- Workaround: a manual reinstall on the failing host, which adds to the drift. The owner then moves P1 to Known error.
- Sub-issues: two staged Defects from tracker rows, and a new Chore that pins one build for every host and account.
- Dependency: a third staged Defect already has a parent, so P1 tracks it as a "blocked by" dependency. P1 waits in Known error until that Defect closes too.
- Related, closed as duplicate: a tracker row in the private operations repository that records a second instance of the drift.
- Resolved when every host and account reports one build on every run in the window and the Resolution criteria pass.

**P2: rows that are really Chores.** Staged items in the rules repository become the Problem "macOS continuous integration (CI) legs fail or queue on shared hosted runners." Refinement confirms the cause, the shared hosted macOS runners, and records a rerun of the leg as the Workaround.

- The owner moves P2 to Known error.
- One staged item is the recurrence check itself. It becomes the Recurrence check section and closes as completed.
- A second staged item becomes a Chore sub-issue whose outcome decides whether the macOS leg blocks a merge.
- Tracker rows that repeat the failure close as duplicates, and their dates go under Related incidents.
- When the Chore closes, the Problem workflow moves P2 to Verifying. The workflow closes it as Resolved if the tar write error stays away for the whole window.
- If the Chore instead records a decision to make the leg non-blocking, the owner closes P2 as Accepted from Verifying, citing that decision with a review date.

**P3: an unknown cause.** The Problem "Scheduled or queued work silently stalls" stays in Investigating. Candidate causes: (1) the schedule trigger stopped, (2) the run fired without writing a record, or (3) no alert reports either case.

- Sub-issue: a new Chore that alerts when a scheduled run misses its window, because every candidate needs it.
- Related only: two tracker rows in different repositories that report schedules not firing. Each becomes a sub-issue once evidence ties it to the confirmed cause.
- P3 stays in Investigating while Cause lists candidates, even after the alert Chore closes. When evidence rules out all candidates but one, the owner rewrites Cause as that root cause and moves P3 to Known error.
- Resolved once the alert has fired zero times across the whole window.

**One item.** A staged item in the rules repository becomes the Problem "Diagram proof fails on some build hosts."

- Refinement confirms a stale cache on the host as the cause, and the owner moves the Problem to Known error.
- Sub-issues: a Chore for the host runbook step and a Defect so the proof refreshes the cache itself.
- Resolved when both merge and the proof passes on each affected host for the window.

**A rejected item.** A staged Defect in the tools repository stays a Defect, without a Problem link. Its line describes one deterministic failure with one code fix, and nothing recurs. The other rejected lines keep their staged types for the same reason.

## Migration from the tracker

The importer (planned) is a one-time job that the records role runs through the content kernel. The tracker lacks a cause column. For that reason the importer leaves Cause empty and opens every Problem it creates in Investigating, and the owner writes the Cause at refinement.

| Tracker column | Problem field |
| --- | --- |
| ID | Body metadata `legacy_id`, because the shared title check refuses an ID prefix |
| Problem | Title and Summary |
| State | Problem status Investigating, with the row state kept as described below |
| Owner | `role:<name>` label |
| Occurrences | Dated lines under Related incidents. The importer leaves past Incidents unopened |
| Last check | A dated history line under Recurrence check |
| Hotfix | Workaround |
| Lasting fix | A Defect or Chore sub-issue when not yet built, or a Resolution criterion when landed |

| Row state | Imported as |
| --- | --- |
| `OPEN` | Investigating |
| `UNCHECKED` | Investigating, with the check under Recurrence check and an empty last-run line |
| `NEVER RUN` | Investigating, with a Chore sub-issue that schedules the check |
| `HOLDING n/m` | Investigating, with the n passing runs as dated history lines |

No row state maps to Known error or Verifying, because no row meets the Known error rule without a confirmed Cause. Runs from before the import never count toward the window, which starts when the Problem enters Verifying.

Each row's executable check remains in its current repository, and the Problem cites it under Recurrence check. A row without a check imports with a Chore sub-issue that writes one. That Chore is a tracked fix, and the Problem workflow moves the Problem to Verifying only after it closes.

A row with one code fix and no recurrence imports as a staged Defect in the responsible repository instead of a Problem. A configuration or runbook row imports as a Chore under a Problem or an Epic. Where a staged issue already exists for a row, the importer converts it to a Problem in place, so its history and links move with it.

## Requests and decisions

A Problem that needs something no agent account can do files an operator command Request, such as placing a credential on a host. The Problem lists the Request under Next step as a relationship, because a Request follows its own states in [operator requests](D-operator-requests.md). A choice between fixes that loosens a trust boundary, such as relaxing a guard, files an approval Request.

That Request's Decision record then backs the move to Accepted or the chosen fix.

## Evidence

| Claim | Status | Source |
| --- | --- | --- |
| The Problem template has Summary, Related incidents, Suspected cause, and Next step | Existing | The Problem template in the private records repository, merged |
| A category list exists | Existing | The record schema in the private records repository, merged |
| The shared `#Criterion` item exists | Existing | `schemas/issue.cue`, tbhb-dev/tools, merged `08c15b8` |
| The shared title check refuses an ID prefix | Existing | `schemas/issue.cue` `#TitlePrefix`, tbhb-dev/tools, merged `08c15b8` |
| Every agent acts as one GitHub user | Existing | The outputs table in [agent roles](../agent-roles/README.md) |
| sm keeps the `role:<name>` labels, checked against a list | New | [The pm design](../pm/README.md) |
| Only pm's sync job writes Status | New | [The pm design](../pm/README.md) |
| A Chore requires a parent | New | [pm content model](../pm/A-content-model.md) |
| A records workflow takes its token from the records app | New | [Operator requests](D-operator-requests.md) |
| The records app can read issues in every repository with a tracked fix | Unverified | This note |
| The Problem variant, its sections, and Problem status | New | This note |
| The Problem workflow and the recurrence check job | New | This note |
| The importer | New | This note |
| Sub-issue limits, same owner, and one parent | Existing | GitHub's sub-issues documentation and REST reference |
| GitHub has no close roll-up | Unverified | GitHub documents none |

## Sources

Each source below reflects a reading on 2026-11-22.

- [GitHub, adding sub-issues](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues) gives 100 sub-issues per parent, eight levels, and links across repositories. It describes no close roll-up.
- [GitHub, REST sub-issues](https://docs.github.com/en/rest/issues/sub-issues) limits sub-issues to repositories of the same owner and gives each issue one parent.
- [GitHub, issue dependencies](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/creating-issue-dependencies) describes the "blocked by" link and the Blocked mark. The link serves a fix that already has a parent. The mark explains why an Incident never becomes a dependency of its Problem.
- [IT Process Maps, problem management](https://wiki.en.it-processmaps.com/index.php/Problem_Management) supports the known-error database, because the Information Technology Infrastructure Library (ITIL) keeps known errors where incident handlers can find them.
- [Splunk, ITIL problem management](https://www.splunk.com/en_us/blog/learn/problem-management.html) supports Known error without a required workaround. Its account of ITIL 4 error control also supports an Accepted close with a review date, a reopen path, and a close for a retired component.
- [Atlassian, documenting known errors](https://support.atlassian.com/jira-service-management-cloud/docs/how-to-document-known-errors-in-a-confluence-knowledge-base/) marks a workaround as temporary or permanent, which matches Known error and Accepted.
- The [Atlassian default problem workflow](https://support.atlassian.com/jira-service-management-cloud/docs/what-is-jira-service-managements-default-problem-workflow/) includes investigation, workaround, and known-error activities, which match Investigating and Known error.
- [ServiceNow, problem management process](https://www.servicenow.com/docs/r/yokohama/it-service-management/problem-management/c_ProblemManagementProcess.html) keeps Known error as a state and closes a problem once its fixes close. This design adds the recurrence check on top.
- [ServiceNow Community, Known error as a state](https://www.servicenow.com/community/itsm-forum/why-was-known-error-removed-as-a-state-on-the-problem-record/m-p/2488764) offers nothing that argues for dropping Known error as a state.
- [Linear, parent and sub-issues](https://linear.app/docs/parent-and-sub-issues) offers an optional parent auto-close. Verifying replaces it with a checked close.
- [Google Site Reliability Engineering (SRE) book, postmortem culture](https://sre.google/sre-book/postmortem-culture/) asks that follow-up fixes get priority, which the Priority field records.
- [Google SRE workbook, postmortem culture](https://sre.google/workbook/postmortem-culture/) gives each tracked action item an owner and a priority, and asks for a measurable end state, which Resolution criteria hold.
