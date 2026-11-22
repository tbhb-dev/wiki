---
type: Design Appendix
title: Service-management records decisions
description: The open and decided questions of the service-management records design.
tags: [service-management, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Service-management records decisions

Back to [the design](README.md).

## Open

None.

## Decided

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
