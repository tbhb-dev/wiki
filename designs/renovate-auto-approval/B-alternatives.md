---
type: Design Appendix
title: Renovate auto-approval alternatives
description: The options the Renovate auto-approval design doesn't take, and why.
tags: [renovate, approvals, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Renovate auto-approval alternatives

Back to [the design](README.md).

## Drop the approval requirement for Renovate

A ruleset bypass for the Renovate App, or a second ruleset without the approval rule, would let pin-only pull requests merge on CI alone. It would also let every Renovate pull request through, pin-only or not, because the ruleset can't see the diff's shape. The operator's direction keeps the approval requirement ([D1](decisions.md)).

## Let Renovate approve its own pull requests

Renovate's identity could post the approval. That makes the author its own approver, which [Architecture Decision Record (ADR) 0009](../../adrs/0009-the-reviews-app-posts-approvals-for-an-independent-approver.md) rules out for reviews, and GitHub doesn't let an author approve its own pull request. The operator's direction names a dedicated App instead.

## Use the reviews App

The reviews App already posts approvals for an independent approver. Its approvals stand for a reviewer's verdict on a diff. A mechanical pin check is a different claim, and mixing the two in one identity would make the review history ambiguous. A separate App also keeps the blast radius small: revoking it stops pin approvals and nothing else.

## Trigger on `pull_request_target`

It would run on every push to the pull request, before CI finishes. The job would then poll for checks, holding a runner and the secret for many minutes, or exit and rely on a second event, which brings back `workflow_run` anyway. It's also the trigger most often misused to run untrusted code with secrets. [Appendix D](D-workflow.md#the-trigger) compares the triggers.

## Trigger on `check_suite` or `check_run`

GitHub doesn't fire either event for a check suite that GitHub Actions created ([appendix A](A-evidence.md)). Every required check here comes from Actions, so neither event would ever fire.

## Run as a webhook service

A small service that receives `check_suite` webhooks could hold the App's key and decide. It would need a host, uptime, and its own secret storage, and the repositories have no such service for this today. A reusable workflow runs on runners the repositories already use and keeps its log beside the run.

## Use the API's diff

`GET /repos/{owner}/{repo}/pulls/{number}` with the diff media type returns a unified diff, but with three context lines and a size limit, and the files endpoint gives per-file patches without mode changes. Git's own diff with zero context, renames, and modes gives the matcher every fact it refuses on. The design keeps the files endpoint as a cross-check ([D6](decisions.md)).

## A general diff classifier

A broader classifier could accept any change that "looks like" a version bump, for example any line where only digits changed. It would accept changes such as a timeout or a port number. The design's grammars accept only named pin shapes in named files, and refuse the rest.

## Approve on Renovate's metadata

Renovate's pull request body lists each update and its type. Trusting it would make the approval depend on text the author wrote. The matcher reads the diff itself and never parses the body.
