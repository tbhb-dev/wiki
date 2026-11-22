---
type: Design Appendix
title: Service-management records operator requests
description: How a Request that only the operator can act on is opened, answered, run, and closed, and how the operator's commands and the workflow keep that safe.
tags: [service-management, requests, operator, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Service-management records operator requests

This is a draft. Nothing here is built yet, and each claim is labeled new, existing, or unverified where it matters. The private records repository holds the records and issues, and a plugin of the shared command-line tool writes them. This document describes both by role.

## Summary

A Request is an issue that asks someone for a decision, an action, access, or information. Most Requests go to an agent role. A Request goes to the operator only when no agent account can do the thing, and its body says why.

The operator answers from their own terminal with four commands: `request run`, `request approve`, `request deny`, and `request comment`. Each posts a comment from the operator's own GitHub login (`run` posts two, one before and one after the script), and a workflow in the private repository reacts only to slash comments from that login. Nobody runs a command for the operator inline in a chat.

## Kinds of Request

Two kinds go to the operator. An approval Request asks for a decision, and the answer becomes a Decision record. A command Request asks the operator to run one prepared script that no agent account may run.

Every Request has a Handler field: Operator or Agent. Agent is the default and the operator never sees it. Operator is allowed only when no agent account can do the work, because a credential, a host, or a one-way door is out of reach, and the Request's body must name that reason in one line.

A Request about information, or a plain ask an agent role can answer, stays with that role. A decision between two reversible options is also not an operator Request: the agent role whose capabilities fit decides and records it. The operator sees one-way doors, credentials, grants, loosened trust boundaries, and their own standing rules.

## States

A Request has one state at a time, shown by a label and moved by the workflow, never by hand.

| State | Meaning | Entered by | Left by |
| --- | --- | --- | --- |
| open | Filed, or edited while open or waiting on author, and waiting for the content check | The open command, or an edit by the agent account in open or waiting on author | The workflow, when the content check passes, or the author closing it |
| waiting on operator | Ready for the operator to act | The workflow, after the content check passes | `/approve`, `/deny`, `/start`, a plain comment, or an author edit, which returns it to open |
| waiting on author | The operator asked for more information | `request comment` | The author edits the body, which returns it to open |
| approved | The operator said yes to an approval Request | `/approve` | The workflow closing it |
| denied | The operator said no, for either kind | `/deny` | The workflow closing it |
| running | The operator started a command Request | The workflow, on the `/start` comment that `request run` posts before it executes | `/complete` |
| complete | The command finished with exit code zero | `/complete` with status success | The workflow closing it |
| failed | The command exited nonzero or stopped partway | `/complete` with status failure | The author opening a new Request, or `/start` from a new `request run` |
| closed | Final, with a recorded outcome | The workflow | Nothing |

Approved, denied, and complete all close the issue with their outcome as the close reason. Failed stays open until a new attempt or an explicit close, so a half-run command is never hidden. The operator decided this (Q2).

Only the operator's login can send `/approve`, `/deny`, `/start`, or `/complete`. The author, which is an agent account, can edit its own Request body while the Request is open, waiting on operator, or waiting on author. The edit sends it back to open for the content check and then to waiting on operator. An edit in running, failed, approved, denied, complete, or closed changes no state, and `run` refuses a body that changed after the operator's last look. The operator decided this (Q1). Any other commenter's slash line is ignored and logged.

## The operator's commands

All four commands use the operator's own `gh` login, not an agent account, so the audit trail shows a human acting. Each command first reads the issue and refuses when the Request is in a state that doesn't allow it.

- `request approve <n> [comment]` posts `/approve` with the optional comment, for approval Requests.
- `request deny <n> [comment]` posts `/deny` with the optional comment, for either kind. A command Request can be denied.
- `request comment <n> <comment>` posts a plain comment and moves the Request to waiting on author. It is the way to ask for more information.
- `request run <n>` runs a command Request and posts the result. The next section gives the detail.

There is no block of copyable commands at the end of each Request. It would need a second write on every Request, because the issue number isn't known until after the issue exists. The operator runs the commands from memory and from `--help`.

A reader who finds a Request in a notification gets the number from the link. The commands take the number alone, so the operator never has to paste a long argument.

## How `request run` works

`request run <n>` executes text that came from an issue, on the operator's machine. That is the largest risk in this design, so the command stops at each check below before it runs anything, and each stop prints the reason.

1. **Fetch.** It reads the issue and its edit history through the API, with the operator's login.
2. **Eligibility.** The Handler must be Operator, the kind must be command, and the state must be waiting on operator, or failed for a retry.
3. **Integrity.** The body must not have been edited after it was opened by anyone but the agent account. The issue's creator must also be the agent account. The edit history gives the editor of each revision, and the command compares the final text with the text the agent account last wrote. A single edit by another account, or an edit after the operator's last look, refuses the run.
4. **Script hash.** The Request's metadata records the SHA-256 hash of the script file, taken when the Request opened. `run` reads the script file once into memory, hashes those bytes, refuses on a mismatch, and later executes those same bytes, so a file replaced after the check never runs. It reads the hash from the first revision of the body, not the latest, so an edit can't swap the script and its hash together. The operator decided this (Q4).
5. **Shape.** The body must hold exactly one command, and that command must be one line. The CUE variant checks this when the Request is opened, and `run` checks it again against the same pattern so a hand-edited body never passes.
6. **Show and confirm.** It prints the Request's title, the stated reason, where to run it, the six one-line fields, and the exact command, then the full text of the verified script bytes held in memory. It asks `y/N` and treats anything but `y` as no.
7. **Execute.** It records the start time, runs the one script with the operator's environment, and captures the exit code, the duration, and the last lines of output.
8. **Report.** Before it executes, it posts a `/start` comment, and after it ends it posts a `/complete` comment with the exit code, the duration, and a short tail of the output.

Before posting, the command redacts anything that looks like a secret. The operator decided that the comment carries a capped, redacted tail, and logged best-effort redaction as a known risk (Q3). A match is replaced with a marker, and the comment says that a line was removed. The output tail is capped at a few lines and a few hundred characters, so a script can't flood the issue. Full output stays on the operator's machine.

If the script exits nonzero, the comment says failure and the Request moves to failed. If the command is interrupted before the script ends, `run` still posts `/complete` with status failure and the partial tail, so the issue never sits in running with nobody watching. If the post itself fails, for example when the network is down, `run` prints the exact comment it would have posted and exits nonzero, and the operator can post it again with the plain GitHub CLI.

A second `request run` on the same Request is allowed only from failed. Running a complete Request again is refused: file a new Request.

## The workflow side

The private repository's workflow reads issue comments. It acts only when the commenter's login equals the operator's login, which is held in the repository's records configuration and changed only by a reviewed change. Comments from any other account, including an agent account, can't change a state. The one other trigger is an edit by the agent account in open, waiting on operator, or waiting on author, which returns the Request to open for the content check. The workflow ignores edits in every other state. The operator decided that the login lives in a code-owned repository configuration file (Q5).

| Comment | Workflow does |
| --- | --- |
| `/approve` | Checks the Request is waiting on operator and an approval Request, labels it approved, closes it as completed |
| `/deny` | Checks the state is waiting on operator, labels it denied, closes it as not planned |
| `/start` | Checks the state is waiting on operator, or failed for a retry, and labels it running |
| `/complete` | Checks the state is running, reads only the first line for the status (the output tail below it is never parsed), labels complete or failed, closes it when complete |
| plain comment from the operator | Moves waiting on operator to waiting on author, and does not close |

The workflow edits only labels and the open or closed state, never the issue body. It uses the API only, runs no code from the issue, and takes its token from the records application the repository already uses.

Agents learn of a result through the event relay the sessions already use. The author's session subscribes to issue and comment events on its own Request, and a state label change wakes it. An agent never polls. If the author has ended, an agent role with the matching capability label picks up the Request.

A `/complete` comment is a record, and the workflow copies its status into a state label. When the operator's reply asks for work, the author opens a follow-up Request or Change rather than reusing a closed one.

## What a command Request contains

A command Request holds one script and nothing else that runs. The script lives in the operator's user-level script directory, named `YYYY-MM-DD-<role>-<slug>.sh` or `.py`, with an optional `.vN` before the extension for a revision. The command line is `bash` or `python3` followed by that path and at most four plain arguments.

The body has these sections, in this order, each one line or one short list:

| Section | Holds | Limit |
| --- | --- | --- |
| Changes | What the script changes | 20 words |
| Checks first | What the operator should check before running | 20 words |
| Exit codes | What each exit code means | 20 words |
| Undo | How to reverse it, or that it can't be reversed | 20 words |
| Dry run | How to see the effect without making it, or why none exists | 15 words |
| Why not an agent | The reason no agent account can run it | 15 words |
| Command | One fenced `bash` block holding the one command line | 1 line |

The whole body is capped at 80 words, and its metadata records the script's SHA-256 hash. Where to run it, here or in the operator's own terminal, is metadata and not a section. Pasted output never goes in the body. It goes in a comment, which the note variant caps.

Section keys, caps, and the command pattern come from a CUE variant. The kernel normalizes the Markdown into blocks and metrics, and the variant constrains them. A body that omits a section, adds an extra block, writes a second command, or names a path outside the script directory is refused with the section and rule named. The variant also requires Handler Operator and kind command as literals, so a body can't pass without them.

The script itself is reviewed before the Request opens. The Request names it, and the script's own file is the artifact the operator reads, not a link to code the agent controls after the fact.

## Threats and debt

| Threat | Effect | What handles it | Status |
| --- | --- | --- | --- |
| Command text from an issue runs on the operator's machine | Arbitrary execution as the operator | Eligibility, integrity, shape, and confirm checks; one script from a reviewed directory | New, unverified until built |
| Edit race: the body changes between the operator's read and the run | The operator runs text they didn't read | The command compares edit history at fetch time and again just before confirm | New |
| Spoofed slash comment from another account | A false approval or completion | The workflow honors only the operator's login, and the login is held in reviewed configuration | New |
| Agent account edits the body after opening | A quiet change of what runs | The agent may edit, but only the final text, as shown to the operator, runs, and the confirm step prints it | New |
| Output carries a secret into a public record | Credential exposure | Redaction, a size cap, and keeping full output local | New, redaction is best effort |
| Operator login compromised | Everything above | Outside this design: the operator's account controls | Existing |
| An agent files Operator Requests to avoid doing work | Operator time spent | The required reason, and reviewers who refuse a reason an agent could meet | New |

These guardrails are deferred and logged as debt: a second reviewer for the script before it opens a Request, a signed hash of the whole body shown at confirm and checked by the workflow, retention limits on the completion comments, and a dashboard of long-waiting Requests.

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | The command-request CUE variant and its fixtures, including the script hash field | A test refuses each hole in the shape and passes a good body | Records role |
| 2 | The workflow's slash-comment handling and state labels | A test run shows an operator comment moves state and another account's comment doesn't | Records role |
| 3 | `request approve`, `deny`, and `comment` | Each posts the right comment against a stub and refuses a wrong state | Records role |
| 4 | `request run` with every check | A test per refusal, and one run end to end against a stub issue | Records role |
| 5 | The first live Request, a harmless script | The issue closes complete and the comment has a status and a tail | Coordination role |

## Questions for the operator

The operator answered all five as recommended. Each answer sits under its question, and the text above already follows them.

### Q1. May the agent account edit a Request body after opening?

- **Recommendation:** Yes, but a changed body goes back to waiting on operator and shows the operator the final text at confirm.
- **Why:** Agents fix typos and caps often, and refusing every edit would make the agent file a new Request each time.
- **If yes:** The integrity check allows edits by the agent account only.
- **If no:** The agent opens a new Request for each fix, and the old one closes as superseded.
- **Reversible:** Yes, by a change to the check.
- **Answer:** Q1: yes. The agent account may edit a Request body after opening, and `run` shows the final text at confirm.

### Q2. Should the workflow close a failed Request on its own?

- **Recommendation:** No. It stays open until a new attempt or an explicit close.
- **Why:** A failed command may have changed something, and a closed issue hides that.
- **If yes:** Failed closes with a failure label, and a retry needs a new Request.
- **If no:** The operator, or an agent role the Request's labels suggest, closes it after reading the tail.
- **Reversible:** Yes.
- **Answer:** Q2: no. A failed run doesn't close its Request on its own.

### Q3. Is a redacted tail of the output enough, or should the comment carry only status?

- **Recommendation:** Keep a capped tail with redaction.
- **Why:** The tail usually says why a run failed, which saves a round trip.
- **If yes:** Redaction is best effort and a known risk, logged as debt.
- **If no:** The comment holds the exit code and duration only, and the operator pastes anything else by hand.
- **Reversible:** Yes, and a leaked line can be edited out of the comment.
- **Answer:** Q3: yes. The completion comment carries a capped, redacted output tail, and the risk is low because the records repository is private.

### Q4. Should `run` refuse when the script file differs from the one named at opening?

- **Recommendation:** Yes, by recording the script's hash in the Request and comparing at run.
- **Why:** The body names a path, and the file behind the path can change after the Request opens.
- **If yes:** The opener records a hash, and `run` refuses on a mismatch.
- **If no:** The operator relies on reading the script at confirm.
- **Reversible:** Yes. This costs one metadata field and one comparison.
- **Answer:** Q4: yes. The Request records the script's hash and `run` refuses on a mismatch.

### Q5. Does the operator's login belong in repository configuration or in an organization-level setting?

- **Recommendation:** Repository configuration, changed only by a reviewed change.
- **Why:** It keeps the check in one reviewed file and needs no new permission.
- **If yes:** The file is a code-owned path.
- **If no:** An organization variable holds it, which needs the admin role to change.
- **Reversible:** Yes.
- **Answer:** Q5: yes. The operator's login lives in a code-owned repository configuration file.
