---
type: Decision Record
title: Automated review can approve delegated permission increases
description: Deploy policy allows automated approval for class R permission increases and reserves class D increases for the operator.
tags: [approvals, permissions, deployment]
status: stable
decided: "2026-11-12"
---

# Automated review can approve delegated permission increases

## Context

The deploy policy distinguishes permission changes that can proceed under standing authority from those that require the operator to decide.

## Decision

Automated review can approve permission increases that deploy policy assigns to class R. The operator approves class D increases. The standing authorization follows the policy’s classification of the change.

## Consequences

A delegated widening can proceed through its normal review without a separate operator request. A recorded decision can revoke the standing authorization.
