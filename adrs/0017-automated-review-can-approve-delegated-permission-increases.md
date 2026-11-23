---
type: Decision Record
title: Automated review can approve delegated permission increases
description: Automated review may approve routine permission increases within the limits set by deploy policy.
tags: [approvals, permissions, deployment]
status: stable
decided: "2026-11-12"
---

# Automated review can approve delegated permission increases

## Context

Deploy policy sets the review rules for changes to agent permissions. Its classes group changes by the authority they need. Standing authorization is ongoing permission to approve qualifying increases. The operator is the human with authority to approve permission increases. Class R covers only the single action the account was denied, on the machine running the agent, for the duration of that action. Requests for a wider scope, another machine or a longer duration go to the operator. Class D is an increase authorized by a decision record already on the default branch at the pull request’s base commit, the commit the proposal builds on. Class D also includes new or changed decision records.

## Decision

The standing authorization follows the deploy policy’s classification of the change. It covers automated approval of class R permission increases and leaves approval of class D increases to the operator.

## Consequences

A class R increase can proceed through normal review without a separate operator request. A recorded decision can revoke the standing authorization.
