---
type: Reference
title: Model roster
description: Which model each kind of work uses and how the roster changes.
tags: [models, roster]
status: draft
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:c581062e61e4c416
    at: "2026-11-22T23:30:54Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model roster

This page says which models tbhb-dev runs and what each suits. Read it before picking a model for a session, an agent, or a peer run. State the model by name on every agent call.

## Standing rulings

- Planning and design run on Opus 5.5 or Fable 5.2 at medium effort, never on Sonnet.
- Sonnet is the default model for Claude Code sessions and Claude subagents, and it stays their fallback.
- Haiku is never the reviewer of record.
- Codex is the default harness for peer agents. Reserve Claude for peer work Codex can't do, such as work that needs an operating-system sandbox.
- Codex is the default for building. Reserve Claude builds for the gaps in the Models table.
- Codex uses `gpt-6.2-sol` for builds and everyday Codex reviews whether the trial runs or not. Outside the trial, Codex supplies the second model family for code review. The reviewer of record checks every finding.
- Review depth follows the tier of the diff, as a starting point. Tier 0 covers changes limited to prose in the wiki, drafts, research, and spikes repos, and needs CI alone. Tier 0 excludes design PRs under `designs/` and decision records under `adrs/`. Tier 1 is most code, tests, refactors, and docs in code repos, and gets one fresh review run. Tier 2 is anything that deletes data or runs destructive commands, touches credentials, rulesets, or permissions, changes agent harness config, deploys, or changes the review and merge path, and gets two review runs. After a fix, one re-review covers the delta at any tier. The paths and content a diff touches set the tier. A reviewer may raise a tier, and nothing lowers it.
- Pick the reviewer model by matching tier. Prefer a model of the same tier on a different provider from the build, using the Matching tiers table. When the build's provider is the last with capacity, use that provider one tier up. A Sonnet build gets an Opus reviewer. The preceding pairing ruling gives the concrete pairs, and this ruling sets the general choice.
- Trial of Codex builds and Opus reviews. Builds go to Codex `gpt-6.2-sol`, and review runs go to Claude Opus 5.5 at medium effort, because Claude has two Ultra plans of quota against one Codex Pro plan. Builds that Codex can't do stay on Sonnet, with Codex `gpt-6.2-sol` reviewing them. When a Codex review run quotes the provider's refusal of security-test code, a Claude review run on Sonnet replaces it. The trial replaces the Antigravity-first and Argon-reviewer lines. Once its window resets, Antigravity uses `gemini-4-hydrogen` for checks without edits. The trial's fit is unverified until [OBSERVATIONS.md](OBSERVATIONS.md) records results.
- Opus 5.5 writes designs, and one Codex `gpt-6.2-sol` run at high effort reviews each design PR. Codex uses `gpt-6.2-astra` for the fallback and hard tier 2 reviews. The harness accepted `gpt-6.2-sol` on 2026-11-26.
- Antigravity runs top-level on the lab hosts. The lab hosts lack the credentials to push. The owner's session replays a run's commit on its own branch through its own gate, then reviews and commits it.
- Planning and design work, in a session or a subagent, follows the planning ruling: the coordinating session runs on Fable 5.2, and every other planning or design session or subagent runs on Opus 5.5 or Fable 5.2. Every other Claude session and subagent runs on Sonnet, except the Haiku jobs in the Models table.
- Any API spend needs @tbhb's approval first.

## Models

| Job | Models and harnesses | Don't use |
| --- | --- | --- |
| Planning and design | Opus 5.5 or Fable 5.2 at medium effort. One Codex `gpt-6.2-sol` run at high effort reviews a design PR | Sonnet or Haiku |
| Building | Codex `gpt-6.2-sol` with or without the trial. Reserve Claude for builds Codex can't do, such as work that needs an operating-system sandbox. When Claude builds, it runs on Sonnet | Opus 5.5 or Fable 5.2 for routine edits that a cheaper model handles |
| Code review | Codex `gpt-6.2-sol` for everyday Codex reviews. In the trial, Claude Opus 5.5 reviews Codex builds at medium effort, and Codex `gpt-6.2-sol` reviews builds that stay on Sonnet. A Claude review run on Sonnet replaces Codex when its provider filter refuses the code. The reviewer of record checks every finding and signs | Haiku as the reviewer of record |
| Checks and scans without edits | Antigravity `gemini-4-hydrogen` once its window resets, with its results unverified. Until then, Codex or Sonnet. Sonnet for tracker checks against main. Haiku for log scans that a person or Sonnet confirms | Haiku alone on checks where a miss matters |
| Coordination sessions | Claude Code on Fable 5.2 for the coordinating session. Planning and design sessions and subagents follow the planning row. Sonnet for every other Claude session and subagent. Codex and Antigravity are unverified for this job | Sonnet for planning or design inside the session |
| First-pass triage | Haiku, and a person or Sonnet confirms its result. Antigravity `gemini-4-hydrogen` once its window resets, unverified | Haiku deciding whether a change is on main from commit subjects alone |

## Matching tiers

Use this table to find the matching reviewer on another provider. Treat a pairing as unverified unless the Support column cites an observation.

| Claude tier | Codex | Antigravity | Support |
| --- | --- | --- | --- |
| Opus 5.5 or Fable 5.2 | `gpt-6.2-astra` for the fallback or a hard review, otherwise `gpt-6.2-sol` at high effort | `gemini-4-argon-high`, which draws down the Antigravity limit the way Fable draws down Claude's | Unverified |
| Sonnet | `gpt-6.2-sol` | No match chosen. `gemini-4-hydrogen` is the cheaper Antigravity model | [OBSERVATIONS.md](OBSERVATIONS.md) records `gpt-6.1-sol` reviewing Claude's code with success, and Antigravity reviewing scorer and launcher code with mixed results. The `gpt-6.2-sol` pairing is unverified |
| Haiku | None named | None named | Unverified |

## Reviewer in the trial

| Build | Reviewer | Support |
| --- | --- | --- |
| Codex `gpt-6.2-sol` | Claude Opus 5.5 at medium effort | Unverified |
| Sonnet, where Codex can't build | Codex `gpt-6.2-sol` | [OBSERVATIONS.md](OBSERVATIONS.md) records `gpt-6.1-sol` reviewing Claude's code with success. The `gpt-6.2-sol` pairing is unverified |
| Either, when the reviewer's provider is out | The build's provider one tier up | Unverified |

Claude models share one subscription's quota, and Codex runs on its own subscription. Neither bills per token, and how Antigravity bills is unverified. The Claude Ultra plans have a weekly window and a separate weekly Fable window, and no 5-hour window, as the usage view shows. Codex has a weekly window. The Antigravity and Codex window claims come from the owners' reports and are unverified, as is the exact quota for each model.

Antigravity keeps two quota pools, one for Gemini models and one for Claude and GPT models. Each pool has a 5-hour meter and a weekly meter. [Google's plans post](https://antigravity.google/blog/changes-to-antigravity-plans) says usage draws down by API price. A cheaper model stretches further, though the post predates Gemini 4. Third-party price lists rank Hydrogen as the cheapest Gemini 4 model, Neon costing more than Argon, and Carbon about level with Argon, and one source disagrees on Carbon. Third parties also report a price rise for Argon in December. If usage follows price, Hydrogen is the cheap reviewer, and high effort adds thinking tokens. These claims come from third parties and stay unverified until OBSERVATIONS.md records measured usage. [Google's model page](https://antigravity.google/docs/models) lists which plans include each model.

Codex released two GPT-6.2 variants. It describes `gpt-6.2-astra` as a model for frontier work and `gpt-6.2-sol` as a model for everyday work. The [pricing page for Codex](https://developers.openai.com/codex/pricing) and other sources put `gpt-6.2-sol` at the same API price as `gpt-6.1-sol` and `gpt-6.2-astra` at four times that price. They also give `gpt-6.2-astra` far fewer local messages per 5-hour window than `gpt-6.2-sol`. These price and quota figures remain unverified.

Antigravity also offers Gemini 4 Carbon (low, medium, and high), Gemini 4 Hydrogen (low through extra high), and two Orchestrator keys. They're available but untested, so their fit is unverified. Use `gemini-4-hydrogen` for Antigravity checks without edits until observations say otherwise.

## Routing by quota

Pick the harness by its remaining weekly headroom. Send reviews, log scans, state checks, triage, and other checks without edits to the harness with the most headroom. Once its window resets, Antigravity takes that peer work on `gemini-4-hydrogen`. Keep builds on Codex while it has headroom, and fall back to Sonnet when Codex is out. Every Claude subagent call states its model. That's Sonnet or Haiku, except for planning and design, which run on Opus 5.5 or Fable 5.2.

## Not yet decided

Host ops, console work, grading evaluations, and approver verdicts have no recommendation yet.

## Updating this page

Base every recommendation change on support from [OBSERVATIONS.md](OBSERVATIONS.md) or a ruling, and say `unverified` for any claim without support. Use [TEMPLATE.md](TEMPLATE.md), check the change against [WRITING.md](WRITING.md), and have it reviewed with [REVIEW.md](REVIEW.md).
