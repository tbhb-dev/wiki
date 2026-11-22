<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Runbooks

A runbook is a procedure to follow step by step. It gives the trigger, the exact commands, and what each outcome means, so any reader can run it cold. Runbooks cover two kinds of procedure: operating procedures for hosts and services, and process procedures that say how work gets done.

## How a runbook differs from other documents

A draft in `drafts/` argues for a change that doesn't exist yet. A runbook describes something that works today, and it stays right only while it matches what the hosts and tools do.

A current-state document describes how a system works. A runbook says what to do with the system and links to the current-state document for the setup.

## Layout

Each runbook is one Markdown file under a directory named for the role that runs it, such as `runbooks/coord/restart-sessions.md`. The filename states the procedure. Procedures that span several roles go in `runbooks/process/`.

A helper script for a runbook is a real file in the same directory, so any host with this repository cloned has it.

## Writing and reviewing a runbook

- [TEMPLATE.md](TEMPLATE.md) gives the skeleton and the rule for a private appendix.
- [WRITING.md](WRITING.md) is the author's checklist before review.
- [REVIEW.md](REVIEW.md) is the reviewer's checklist and report format.

A runbook lands only after two reviews from models that didn't write it clear it of blocking items at the exact head they read.

## Public pages and private appendices

This repository is public, so a runbook starts here. A runbook that needs private specifics, such as host names or account details, splits in two: the public page holds the procedure, and a private appendix with the same slug holds the specifics. The public page says that a private appendix exists and links to nothing private. [TEMPLATE.md](TEMPLATE.md) shows the wording.

## Upkeep

Check a runbook against the host or tool it describes before relying on it. Whoever changes the hosts or tools fixes the runbook in the same change, or adds a line above Purpose that says the page is stale.

## Planned checks

A rules check that fails a runbook with missing frontmatter or missing sections is a follow-up for tbhb-dev/rules. It doesn't exist yet.
