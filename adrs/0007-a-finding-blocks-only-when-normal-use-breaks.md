---
type: Decision Record
title: A finding blocks only when normal use breaks
description: A review finding is a BLOCK only when ordinary use with no adversary gives a wrong, destructive or broken result, and other findings become logged debt.
tags: [review, findings, debt]
status: stable
decided: "2026-11-23"
---

# A finding blocks only when normal use breaks

## Context

Review runs held pull requests on findings that needed a hostile process inside the agent account, or on failures that did nothing harmful. Each hold cost another fix round and another review, while the change it held worked for every honest use. The tbhb-dev work runs MVP first, so only a failure of basic use should stop a merge.

## Decision

A finding is a BLOCK only when ordinary use, with no adversary, produces a wrong, destructive or broken result.

A finding that needs a same-user process to race, swap or booby-trap files inside the agent account is SECURITY-DEBT, since the agent account is the boundary.

A failure that fails safe, by doing nothing, refusing or leaving state behind, is TECH-DEBT.

Best-effort text checks, such as a CI-restatement filter or a title-shape check, follow the same rule. A false refusal of honest text is a BLOCK, because it breaks basic use. A missed match is TECH-DEBT, logged with the missed example as a fixture for a follow-up.

Reviewers classify findings this way in their own output. The coordinating role checks every held pull request against this rule before it waits on a fix.

## Consequences

Each SECURITY-DEBT or TECH-DEBT finding becomes a debt row in its owner's tracker, with the example as a fixture, so hardening work keeps the evidence. A reclassification covers only the exact head the reviewer judged. A pull request with only debt findings can merge once its tier's reviews are otherwise GO.
