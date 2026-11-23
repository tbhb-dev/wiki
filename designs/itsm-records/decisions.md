---
type: Design Appendix
title: Service-management records decisions
description: The open and decided questions of the service-management records design.
tags: [service-management, decisions, design]
status: draft
parent: README.md
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:933656c1cea36ed5
    subagent: sha-256-64:ea6e32cf22cbcb34
    at: "2026-11-22T23:15:39Z"
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:933656c1cea36ed5
    at: "2026-11-22T23:15:39Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Service-management records decisions

Back to [the design](README.md).

## Open

None.

## Decided

### D8. Where does the importer put a Problem made from a staged item?

- **Decided:** 2026-11-23, by the service-management maintainer, within the Problem design direction the operator approved.
- **Rule:** Every Problem lives in the private records repository. The importer opens the Problem there, moves the staged item's sub-issues and dependencies to it, and closes the staged item as a duplicate of the Problem, which leaves a pointer.
- **Why:** The Problem workflow runs only in the records repository, so a staged item converted in place elsewhere would never move between states.
- **Undo:** Close the duplicated staged items and run the Problem workflow in the private records repository only.
- **Record:** [Problems](problems.md), the section on migration from the tracker.

### D7. How long does the recurrence window last, and when does it pass?

- **Decided:** 2026-11-23, by the service-management maintainer, within the Problem design direction the operator approved.
- **Rule:** The window lasts 14 days by default and starts when the Problem enters Verifying. It passes only when every run passed, each scheduled period has a recorded run, and the window has at least seven runs. A schedule too sparse for seven runs lengthens the window, and a check that runs at least daily may set a shorter one.
- **Why:** Two weeks suits a daily job, the seven-run floor keeps a dropped scheduled job from passing an empty window, and a weekly job waits seven weeks for its window.
- **Undo:** Change the default length or the minimum count in the design and the Problem workflow. Each open Problem keeps the window it already names.
- **Record:** [Problems](problems.md), the section on the window and its evidence.

### D6. May an agent role close a Problem as Accepted without an approval Request?

- **Decided:** 2026-11-23, by the service-management maintainer, within the Problem design direction the operator approved.
- **Rule:** Yes, on the role's own decision line with a review date. A workaround that loosens a guard or a trust boundary needs an approval Request first.
- **Why:** Any role can reopen an Accepted Problem, so the close is a two-way door, while a loosened guard changes a trust boundary that only the operator grants.
- **Undo:** Require an approval Request before every Accepted close.
- **Record:** [Problems](problems.md), the section on ending without a fix.

### D5. Where does the operator's login live?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** The login sits in a code-owned repository configuration file, changed only by a reviewed change.
- **Undo:** Move it to an organization variable, which needs the admin role to change.
- **Record:** The operator's decision log, entry dated 2026-11-22.

### D4. Does `request run` refuse when the script differs from the one named at opening?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** The Request records the script's hash, and `run` refuses on a mismatch.
- **Undo:** Drop the metadata field and the comparison, and rely on the operator reading the script at confirm.
- **Record:** The operator's decision log, entry dated 2026-11-22.

### D3. Is a redacted output tail enough, or should the completion comment carry only status?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** The `/complete` comment carries a capped, redacted tail of the output. Redaction is best effort, and the risk is low because the records repository is private.
- **Undo:** Post the exit code and duration only, and edit any leaked line out of the comment.
- **Record:** The operator's decision log, entry dated 2026-11-22.

### D2. Should the workflow close a failed Request on its own?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** No. A failed run leaves its Request open until a new attempt or an explicit close.
- **Undo:** Close failed Requests with a failure label, and require a new Request to retry.
- **Record:** The operator's decision log, entry dated 2026-11-22.

### D1. May the agent account edit a Request body after opening?

- **Decided:** 2026-11-22, by the operator.
- **Rule:** Yes. The edited Request returns to waiting on the operator, and `run` shows the final text at confirm.
- **Undo:** Refuse edits, and have the agent open a new Request for each fix.
- **Record:** The operator's decision log, entry dated 2026-11-22.
