---
type: Readme
title: Decision records
description: Which decisions this repository records, the record layout, and the index of records.
tags: [adrs, decisions, index]
status: draft
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:c581062e61e4c416
    at: "2026-11-23T00:26:49Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Decision records

This public repository holds the decision records that can be stated without anything private. An ADR goes here whenever it can be sanitized without losing its meaning: host names, the tailnet, IP addresses, internal paths, account names and credentials are removed or made generic, and the rule still reads the same. Only a record that would lose its meaning goes in the private wiki repository's `adrs/` folder.

A decision record (ADR) states one decision as a rule in the present tense, with the context that led to it and what follows from it. A record makes sense to a reader of this repository alone.

## Layout

Each record is `adrs/NNNN-<slug>.md`: the next free number, zero-padded to four digits, then a kebab-case slug of the title, such as `0007-squash-merge-every-pull-request.md`. This repository numbers its own records from `0001`. [TEMPLATE.md](TEMPLATE.md) is the house template, and the format and writing rules are in the decision-records design.

## Rules

- A record never names a host, the tailnet, a vault, an internal path, an account, a private repository or a public IP address. It uses role labels such as "the Linux build host".
- A record never cites a Decisions note or links to a private repository.
- A record never quotes anyone or says who decided. Acceptance is the pull request's approval and merge, and the pull request cites the source of the decision.
- The `decided` field is the date of the decision, not of the record.
- Nobody renames a record after it merges, with one exception: the fix for a duplicate number renames the later file.

## Records

Add each new record's entry at the end of this list, as `[title](NNNN-slug.md)` and the record's description. When two pull requests add a record at once, the later one rebases and takes the next number.

None yet.
