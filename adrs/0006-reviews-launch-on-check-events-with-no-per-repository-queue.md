---
type: Decision Record
title: Reviews launch on check events, with no per-repository queue
description: A green conclusion check on a held head launches its reviews at once, every open pull request is reviewed in parallel, and only host capacity limits it.
tags: [review, events, concurrency]
status: stable
decided: "2026-11-22"
---

# Reviews launch on check events, with no per-repository queue

## Context

Owners used to send a message to the reviewing session when a head turned green, and reviews waited on that message. Repositories also kept one pull request in review at a time, so an owner waited for each merge before starting the next build. Both made review the slowest step for reasons unrelated to the work.

## Decision

The review launcher subscribes to the success event of the `conclusion` check run in each repository it serves. When an event's head SHA equals a held head, it launches the queued review for that head. Owners don't send green-line messages to reviewers.

When a pull request's only red check is the size step, the review launcher runs the sizing review at once, without waiting for the owner to ask.

No repository has a limit of one pull request in review. Owners keep several pull requests open and start the next build without waiting for the previous merge. Review sessions launch reviews for every green head at once. Review sessions review stacked pull requests in parallel, judging each as its own diff against its base. A stacked pull request needs a new review only when its own diff changes, such as after a fix lower in the stack.

The only limits are host capacity and the launcher's cap on concurrent runs.

## Consequences

Review starts as soon as CI finishes, and a slow merge in one pull request doesn't hold the others. Merge commits keep stacked heads unchanged when a parent merges, so their reviews stay valid. More reviews run at once, so the launcher's run cap and host capacity decide throughput. To undo the event trigger, unsubscribe and go back to green-line messages. To undo the sizing trigger, owners ask for sizing reviews again.
