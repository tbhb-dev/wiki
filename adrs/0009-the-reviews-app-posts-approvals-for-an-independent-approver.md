---
type: Decision Record
title: The reviews app posts approvals for an independent approver
description: The reviews app posts approval after an independent approver verifies the proposed revision and its review evidence.
tags: [approvals, reviews, github]
status: stable
decided: "2026-11-23"
---

# The reviews app posts approvals for an independent approver

## Context

A review verdict is the reviewer’s judgment of the change. A GitHub approval is a separate action that needs its own identity check. The independent approver is separate from the maker, who produced the change, and the opener, who submitted the pull request. The head is the latest commit on the pull request’s source branch. The diff is the set of proposed changes.

This rule governs posting approval for the checked head and leaves the underlying review verdicts to the reviewers.

## Decision

The reviews app is the GitHub integration that posts approvals through `tbhbdev alpha review create`, the command that submits a pull request review. The approver reads the full diff and required review evidence for the exact head. Approval requires a successful `conclusion` check, the summary of required jobs. GitHub must mark every listed commit's signature as verified. The service checks the run record saved locally by the launcher, which starts agent runs. It rejects an approval from the opener’s root identity, the identifier shared by a main session and its child runs.

An approver role can run on demand. The same independence and evidence checks apply to that run.

## Consequences

Until the app can post approvals, the designated approver uses `tbhb-approve`, the existing approval command, with the exact head. The coordination loop, the automation that advances pull requests, can act on a reviewer’s approval only after leaving shadow mode, in which it takes no live action. The loop must confirm that the reviewer read the full diff. It must also confirm that the head passed the conclusion check.
