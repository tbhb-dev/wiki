---
type: Decision Record
title: Repeated change requests trigger a design assessment, not a human handoff
description: Three rounds of requested changes on a pull request don't hand it to a human automatically, but they do trigger an assessment of whether the design or requirements are flawed.
tags: [review, escalation]
status: stable
decided: "2026-11-11"
---

# Repeated change requests trigger a design assessment, not a human handoff

## Context

A pull request can go through round after round of requested changes, each fix opening a new finding. Handing such a pull request to a human after a fixed count would put routine review load on the operator. Repeated rounds often mean the approach itself is wrong, which another fix round can't cure.

## Decision

Three rounds of requested changes don't hand a pull request to a human automatically.

Three rounds do trigger an assessment of whether the design or the requirements behind the change have a fundamental flaw, before more fix rounds go ahead.

## Consequences

The GitHub broker design needs a better escalation path than the automatic handoff, and that path is still open there. In one case, three rounds each found a new leak form opened by a broad exemption, and the fix replaced the exemption with a short allowlist of exact entries instead of patching it again.
