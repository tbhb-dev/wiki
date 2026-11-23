---
type: Decision Record
title: The reviews App approves, never for the session that opened the pull request
description: Approvals come from the reviews GitHub App after a host-side check that the requester didn't open the pull request, and never with an unverified commit.
tags: [review, approval, github-apps]
status: stable
decided: "2026-11-22"
---

# The reviews App approves, never for the session that opened the pull request

## Context

Branch rulesets need an approving review before a merge. Agents act through GitHub Apps, so an approval has to come from an App identity. The maker of a change must never approve it, and a session's own word isn't enough to prove it didn't open the pull request. A pull request also failed to merge after approval because one of its commits carried a signature GitHub didn't recognize, and the default branch requires signed commits.

## Decision

The reviews GitHub App posts each approval, as an APPROVE review created through the tbhbdev review command. It never approves on behalf of the root session that opened the pull request. The check runs on the host side, from the run host's record of who opened it, not from anything the requesting session claims.

An approval needs the `conclusion` check green at the exact head and the GO verdicts that the pull request's tier needs at that head, read from the review runs themselves.

An approval stops when the pull request lists any unverified commit. The author replaces the commit with a signed one, which makes a new head that needs its own review.

## Consequences

Approval rights live in one App with one host-side check, rather than in each session's own discipline. Until the App's review command exists, the approving session's approval command stays the path, under the same conditions. A launched run of the approver kind may make the approver's read and run that command instead of a long-running session. Tier 0 pull requests need no approval, per [0002](0002-tier-0-pull-requests-merge-on-green-without-review-or-approval.md).
