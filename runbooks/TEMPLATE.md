<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Runbook template

Copy the skeleton into `runbooks/<role>/<procedure>.md`, replace the placeholders in angle brackets, and delete the guidance comments. [WRITING.md](WRITING.md) lists the rules the file follows, and [REVIEW.md](REVIEW.md) is what the reviewer checks.

## The skeleton

````markdown
---
type: Runbook
title: <The procedure in plain words, at most 8 words>
description: <One sentence: what this does and who runs it.>
owner: <durable role, never a session>
updated: "<YYYY-MM-DD>"
last_verified: "<YYYY-MM-DD, or never>"   # the last date someone ran it against the live host or tool; never means nobody has
---

# <Title>

## Purpose

<What this procedure achieves, in one or two sentences.>

## Trigger

<The event or condition that starts it. Say when not to run it.>

## Preconditions

- <What must hold first: access, tools, state. Each item is something the reader can check.>

## Steps

1. <What to do, in one sentence.>

   ```sh
   <the exact command, on its own line>
   ```

   <What the output looks like when it worked.>

## What each outcome means

| Outcome | Meaning | Next |
| --- | --- | --- |
| <result the reader sees> | <what it says about the system> | <the next step or runbook> |

## Failure handling and undo

<For each step that can fail, how to tell and what to do. Say how to undo the change, or say that it's a one-way door and why.>

## Related

- <Link to the current-state document, the tracker, or the next runbook.>
````

## Splitting off a private appendix

Write the whole runbook here when nothing in it is private. When the procedure needs private specifics, write the procedure here and keep the specifics in a private appendix with the same slug.

The public page says that a private appendix exists, in one line under Related:

```markdown
- A private appendix holds the host-specific details for this procedure.
```

The public page never names the private repository, a private host, a tailnet detail, a public IP address, or a secret. Steps that need a private value say what the value is, such as "the host's address", and the appendix gives it.

## Process runbooks

A process runbook says how work gets done across roles, such as how a design gets written and reviewed. It uses the same skeleton, lives in `runbooks/process/`, and names each role that acts in each step.
