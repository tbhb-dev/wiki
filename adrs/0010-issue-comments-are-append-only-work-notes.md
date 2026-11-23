---
type: Decision Record
title: Issue comments are append-only work notes
description: A comment on a service-management issue is a typed, append-only work note written from a template, and every state transition writes its own note.
tags: [issues, work-notes, service-management]
status: stable
decided: "2026-11-22"
---

# Issue comments are append-only work notes

## Context

Free-form comments on issues mix updates, evidence, handoffs and answers in no fixed shape. Edited or deleted comments rewrite the history, and nothing records who moved an issue between states or why. ServiceNow's work notes solve the same problem with typed, append-only entries.

## Decision

A comment on a service-management issue is a work note. It's written only through the managed note command, from a template that carries the author's role, a kind and a body. The kind is one of update, evidence, handoff, blocker, decision or answer. Claims in the body carry their evidence. Work notes are append-only: nobody edits or deletes one, and a correction is a new note that refers to the note it corrects. Every state transition writes an automatic note recording who made it, the from and to states, and why.

## Options considered

- Free-form, editable comments: rejected because the history can be rewritten and nothing can be checked against a shape.

## Consequences

An issue's history is complete and readable in order, and handoffs between runs carry a known structure. Fixing a mistake leaves the original visible next to its correction. The content kernel can check each note against its template like any other content type.
