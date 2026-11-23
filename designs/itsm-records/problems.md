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

This is a draft. Nothing here exists as code unless the evidence table cites a merged path. A Problem lives in the private records repository, and its fixes live where the code lives.

## The types

An Incident records one interruption and closes when service returns. A Problem records the cause behind one or more Incidents, behind a recurring failure, or behind a failure nobody can explain. It closes only after its fixes land and a recurrence check passes.

A Defect is one code fix. A Chore is one configuration or runbook fix. A Defect found in code with a clear fix and no recurrence needs no Problem.

## Parent or dependency

**Decision:** a Problem is the sub-issue parent of each Defect and Chore that fixes its cause. An Incident names its Problem in a body line, and the Problem lists the Incident under Related incidents. Neither end uses a sub-issue link or a dependency.

**Rejected:** a "blocked by" dependency from the Problem to each fix because it gives no progress count, and pm requires a parent on every Chore. Also rejected: an Incident as a sub-issue or a dependency of its Problem, because the Incident closes on restore, well before the Problem, and GitHub would mark it Blocked.

**Tradeoff:** GitHub allows 100 sub-issues per parent and eight levels, across repositories of the same owner. An issue takes one parent, so a fix that already sits under an Epic, or under a second Problem, joins this Problem as a "blocked by" dependency instead. GitHub documents no close roll-up in either direction, so the last child closing moves the Problem to Verifying, and only the owner closes it.

## Fields

| Fact | Home | Values | Set by |
| --- | --- | --- | --- |
| Type | Native issue type | Problem | The open verb |
| Problem status | Org issue field | Investigating, Known error, Verifying, Resolved, Accepted | See transitions |
| Source | Body metadata | Incident, Recurrence, Check, Review, Import | The open verb, once |
| Category | Body metadata | The existing category list | The open verb |
| Owner | Assignee | One agent role | Refinement |
| Priority | pm's Project field | Expedite, Standard, Intangible | Refinement, from impact and how often it recurs |
| Area | `area:<repo>` label | The affected repository | The open verb |

The field name differs from pm's Status, which only the sync job writes from GitHub state. Labels lost because two state labels can coexist and nothing on a Problem moves them automatically. Body metadata lost because each move would edit the body and rerun the content check.

| From | To | Who moves it | Condition |
| --- | --- | --- | --- |
| (new) | Investigating | The open verb | Source set |
| Investigating | Known error | The owner | Cause filled, and Workaround filled or marked None |
| Known error | Verifying | The sync job | The last child closed |
| Verifying | Resolved | The owner, closing as completed | The recurrence check passed every run in its window |
| Verifying, Resolved, or Accepted | Investigating | Any role or the check | A recurrence the workaround does not cover, or an Accepted review that finds a fix now worth it, which reopens the issue |
| Investigating or Known error | Accepted | The owner, closing as not planned | A recorded decision to live with the workaround, with a review date, or the affected component retired |

No separate known-error record exists. Open Problems in Known error and closed Problems marked Accepted form the known-error database, and an Incident responder searches them for a Workaround.

The template at `templates/problems/problem.md` in the private records repository changes in four places. Suspected cause becomes Cause, holding either the root cause or a numbered list of candidate causes. New sections follow: Workaround, Resolution criteria (a list of the shared `#Criterion` items), and Recurrence check (one command or query, its expected result, and its window). Summary, Related incidents, and Next step keep their names.

## Refinement

pm's refinement design does not exist in the wiki at this head, so this section defines only the Problem's ready check. It follows pm's hierarchy in [the pm design](../pm/README.md): a Defect or Chore links to its parent as a sub-issue.

A Problem is ready for investigation when it meets four conditions:

- **Scope:** one sentence naming the symptoms and the hosts or repositories in scope.
- **Evidence:** at least one dated Incident, occurrence, or failing check.
- **Owner:** one assigned agent role.
- **Next step:** one action with a time box. A step too large for that line becomes a Spike sub-issue.

Refinement of a staged Defect has three outcomes. It stays a Defect, it becomes the child of an existing Problem, or it becomes a new Problem. Items that merely look alike link as Related until evidence shows one cause.

## Worked splits

Each split quotes the import candidate list, naming a private repository by role.

**P1, a cluster with one cause and a workaround.** tools/problems-r1 becomes the Problem "Launcher builds drift across hosts and accounts", Source Import, status Known error. Cause: hand-installed builds, rolled back per host, leave hosts and accounts on different builds. Workaround: hand-install the matching build on the failing host, which also feeds the drift.

- Sub-issues: tools/problems-r18, tools/problems-r22, and tools/problems-lab-private-mirror-fetch, each a Defect. A new Chore pins one build for every host and account.
- Related, closed as duplicate: standing-directions-p18 in the private operations repository, a second instance of the drift.
- P3 takes the mirror-fetch Defect as a "blocked by" dependency, because the Defect already has P1 as parent.
- Closes when every child closes and every host and account reports one build for the whole window.

**P2, rows that are really Chores.** The Problem "macOS CI legs fail or queue on shared hosted runners" opens with Source Import, status Known error. Cause: shared hosted macOS runners. Workaround: rerun the leg.

- rules/rules-tech-debt-n97 is the recurrence check itself. It becomes the Recurrence check section and closes as completed.
- rules/rules-tech-debt-n35 becomes a Chore sub-issue, and its outcome is a recorded decision.
- rules/dashboard-apps-a10 and the private operations repository's problems-r12 close as duplicates, and their dates go under Related incidents.
- Closes as Resolved when the tar write error stops for the window, or as Accepted once n35 records a non-blocking leg.

**P6, an unknown cause.** The Problem "Scheduled or queued work silently stalls" opens with Source Import, status Investigating. Candidate causes: (1) the schedule trigger stopped, (2) the run fired and left no record, (3) a missing alert lets either go unseen.

- Sub-issue: a new Chore that alerts when a scheduled run misses its window, because every candidate needs it.
- Related only: renovate-config/problems-actions-schedules-not-firing and the private operations repository's problems-r5. Each becomes a sub-issue once evidence ties it to the confirmed cause.
- Moves to Known error when one candidate survives. Closes after the alert reports no missed window.

**A single item.** rules/problems-r9 becomes the Problem "Structurizr proof fails on Linux lab hosts", Source Import, status Known error. Cause: a stale font cache. Sub-issues: a Chore for the host runbook step and a Defect so the proof refreshes the cache itself. Closes when both land and the proof passes on each Linux lab host for the window.

**A rejected item.** tools/guardrails-tooling-94 stays a Defect with no Problem and no link. Its line names one deterministic failure with one code fix, and nothing recurs. The rejected lines keep their staged types for the same reason.

## Migration from the tracker

| Tracker column | Problem field |
| --- | --- |
| ID | Body metadata `legacy_id`, because the shared title check refuses an ID prefix |
| Problem | Title and Summary |
| State | Problem status, as mapped below |
| Owner | Assignee |
| Occurrences | Dated lines under Related incidents. The importer opens no past Incidents |
| Last check | The last-run line of Recurrence check |
| Hotfix | Workaround |
| Lasting fix | A Defect or Chore sub-issue when unbuilt, or a Resolution criterion when landed |

| Row state | Problem status |
| --- | --- |
| OPEN | Investigating, or Known error when Hotfix names a working workaround |
| UNCHECKED | Verifying, with an empty last-run line |
| NEVER RUN | Verifying, with a Chore sub-issue that runs the check once |
| HOLDING n/m | Verifying, with n of m passing runs recorded. Resolved only at m of m |

Each row's executable check keeps its current home, and the Problem cites it under Recurrence check. A row without a check imports with a Chore sub-issue that writes one, and the Problem cannot close before that Chore closes. A row with one code fix and no recurrence imports as a staged Defect in the responsible repository. A configuration or runbook row imports as a Chore under a Problem or an Epic.

## Requests and decisions

A Problem that needs something no agent account can do files an operator command Request, such as placing a credential on a host for P7. The Problem lists the Request under Next step as a relationship, because a Request follows its own states in [operator requests](D-operator-requests.md). A choice between fixes that loosens a trust boundary, such as relaxing a P4 guard, files an approval Request, and its Decision record backs the move to Accepted or the chosen fix.

## Evidence

| Claim | Status | Source |
| --- | --- | --- |
| The Problem template has Summary, Related incidents, Suspected cause, and Next step | Built | `templates/problems/problem.md`, private records repository, merged `60c45db` |
| A category list exists | Built | `schemas/sm.cue` `_category`, private records repository, merged `e7caa1b` |
| The shared `#Criterion` item exists | Built | `schemas/issue.cue`, tbhb-dev/tools, merged `08c15b8` |
| The shared title check refuses an ID prefix | Built | `schemas/issue.cue` `#TitlePrefix`, tbhb-dev/tools, merged `08c15b8` |
| A Chore requires a parent | Unbuilt | [pm content model](../pm/A-content-model.md) |
| The Problem variant, its sections, and Problem status | Unbuilt | This note |
| The sync job's move to Verifying and the close check | Unbuilt | This note |
| The importer | Unbuilt | This note |
| Sub-issue limits, same owner, and one parent | External | GitHub's sub-issues documentation and REST reference, read 2026-11-22. No close roll-up is unverified |

## Open questions for the operator

1. **May an agent role close a Problem as Accepted without a decision Request?** Recommendation: yes, unless the workaround loosens a guard or a trust boundary, which needs an approval Request.
2. **Does the recurrence window default to 14 days?** Recommendation: yes, with a shorter window allowed when the Problem's Recurrence check names a daily run.
3. **Should the import convert tools/problems-r1 in place or open a new Problem?** Recommendation: convert in place, so its history and links move with it.

## Sources

Each source was read on 2026-11-22.

- [GitHub, adding sub-issues](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues): no change, because it confirms 100 sub-issues, eight levels, and cross-repository links, and says nothing about close roll-up.
- [GitHub, REST sub-issues](https://docs.github.com/en/rest/issues/sub-issues): the tradeoff now limits sub-issues to repositories of the same owner, and one parent per issue moves from unverified to cited.
- [GitHub, issue dependencies](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/creating-issue-dependencies): no change, because the Blocked mark it describes is why an Incident is not a dependency of its Problem.
- [IT Process Maps, problem management](https://wiki.en.it-processmaps.com/index.php/Problem_Management): the known-error database line, because the Information Technology Infrastructure Library (ITIL) keeps known errors where incident handlers can find them.
- [Splunk, ITIL problem management](https://www.splunk.com/en_us/blog/learn/problem-management.html): Known error no longer needs a workaround, Accepted gains a review date and a reopen path, and a retired component closes as Accepted, after ITIL 4's error control.
- [Atlassian, documenting known errors](https://support.atlassian.com/jira-service-management-cloud/docs/how-to-document-known-errors-in-a-confluence-knowledge-base/): no change, because its temporary or permanent workaround status matches Known error and Accepted.
- [Atlassian, default problem workflow](https://support.atlassian.com/jira-service-management-cloud/docs/what-is-jira-service-managements-default-problem-workflow/): no change, because the readable text names only investigation, workaround, and known-error activities.
- [ServiceNow, problem management process](https://www.servicenow.com/docs/r/yokohama/it-service-management/problem-management/c_ProblemManagementProcess.html): no change, because Known error as a state and a close driven by the fixes closing match this note, which adds the recurrence check.
- [ServiceNow Community, Known error as a state](https://www.servicenow.com/community/itsm-forum/why-was-known-error-removed-as-a-state-on-the-problem-record/m-p/2488764): no change, because the thread gives no reason to drop the state.
- [Linear, parent and sub-issues](https://linear.app/docs/parent-and-sub-issues): no change, because its optional parent auto-close is what Verifying replaces.
- [Google SRE book, postmortem culture](https://sre.google/sre-book/postmortem-culture/): no change, because it asks only that follow-up fixes get priority.
- [Google SRE workbook, postmortem culture](https://sre.google/workbook/postmortem-culture/): the Priority row, because each tracked action item needs an owner and a priority, and its measurable end state confirms Resolution criteria.
