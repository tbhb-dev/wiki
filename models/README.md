<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model roster

This page says which models tbhb-dev runs and what each suits. Read it before picking a model for a session, an agent, or a peer run. State the model by name on every agent call.

## Standing rulings

- Planning and design run on Opus 5.5 or Fable 5.2 at medium effort, never on Sonnet.
- Sonnet is the default model for Claude Code sessions and Claude subagents, and it stays their fallback.
- Haiku is never the reviewer of record.
- Codex is the default harness for peer agents. Claude runs a peer agent only where Codex can't, such as work that needs an operating-system sandbox.
- Codex is the default for building. Claude builds only for a gap that the Models table lists.
- Codex `gpt-6.1-sol` is the second model family for code review. The reviewer of record checks every finding. Claude reviews only as the signer, except in the two pairings that follow.
- The review pair follows the model that wrote the diff. A diff from a Codex run, or written by the owner's own session, gets Codex and Antigravity as full peers. A diff with any part from an Antigravity run, Neon or Argon, gets Codex and a Claude review run on Sonnet, and an Antigravity review doesn't count. Codex's provider filter refuses some security-test code. When a Codex review run quotes that refusal, a Claude review run on Sonnet replaces the Codex half.
- Review depth follows the tier of the diff, as a starting point. Tier 0 is prose-only changes in the wiki, drafts, research, and spikes repos, and needs only CI. Tier 1 is most code, tests, refactors, and docs in code repos, and gets one fresh review run. Tier 2 is anything that deletes data or runs destructive commands, touches credentials, rulesets, or permissions, changes agent harness config, deploys, or changes the review and merge path, and gets two review runs. A re-review after a fix covers only the delta, with one run, at any tier. The paths and content a diff touches set the tier, never the maker's judgment. A reviewer may raise a tier, and nothing lowers it.
- Pick the reviewer model by matching tier. Prefer a model of the same tier on a different provider from the build, using the Matching tiers table. When only the build's provider has capacity left, use that provider one tier up: a Sonnet build gets an Opus reviewer. The preceding pairing ruling gives the concrete pairs, and this ruling sets the general choice.
- Scale the reviewer model to the tier. Tier 2 may use a top-tier model, which is Antigravity `gemini-4-argon-high` or Codex `gpt-6.1-sol` at high effort, with Opus on the same-provider fallback. Tier 1 and delta re-reviews use a lighter model or lower effort, such as `gpt-6.1-sol` at medium effort, Neon, or a Carbon key. Argon high is a top-tier model, and as the default Antigravity reviewer it used up its 7-day window fast. The tier 2 limit protects that window, and the lighter choices are unverified until [OBSERVATIONS.md](OBSERVATIONS.md) supports them.
- Antigravity runs top-level on the lab hosts. Its runs can't push, because the lab hosts hold no write credential. The owner's session replays a run's commit on its own branch through its own gate, then reviews and commits it.
- During the Antigravity trial, new design and build work goes to Antigravity first, then Codex. Jobs that need shell commands or research go to Codex for now, because headless Antigravity auto-denies shell commands. Claude takes only coordination, review, and work that neither can do. The trial overrides the planning-and-design and Codex-builds lines preceding it. Each owner records the quality of a task in [OBSERVATIONS.md](OBSERVATIONS.md).
- Some builds go to Antigravity `gemini-4-neon-high`, and hard builds to the same model at extra-high effort. Codex stays a builder too. Neon's observations are in [OBSERVATIONS.md](OBSERVATIONS.md), and how well it builds is unverified.
- Planning and design work, in a session or a subagent, follows the planning ruling: the coordinating session runs on Fable 5.2, and every other planning or design session or subagent runs on Opus 5.5 or Fable 5.2. Every other Claude session and subagent runs on Sonnet, except the Haiku jobs in the Models table.
- Any API spend needs @tbhb's approval first.

## Models

| Job | Models and harnesses | Don't use |
| --- | --- | --- |
| Planning and design | Opus 5.5 or Fable 5.2 at medium effort | Sonnet or Haiku |
| Building | Codex `gpt-6.1-sol` by default. Claude builds only where Codex can't, such as work that needs an operating-system sandbox. When Claude builds, it runs on Sonnet | Opus 5.5 or Fable 5.2 for routine edits that a cheaper model handles |
| Building, Antigravity share | `gemini-4-neon-high`, or the same model at extra-high effort for a hard build, for a share of builds. Its commit comes back as a patch that the owner replays and reviews. Its fit is unverified | An Antigravity review of a diff that Antigravity built |
| Code review | Codex `gpt-6.1-sol` and Antigravity `gemini-4-argon-high` (at tier 2, lighter at tier 1) as full peers for the review read. A diff built by Antigravity gets Codex and Claude. A Claude review run on Sonnet replaces Codex when its provider filter refuses the code. How well Antigravity reviews is unverified. The reviewer of record checks every finding and signs | Haiku as the reviewer of record |
| Read-only checks and scans | Antigravity `gemini-4-argon-high` first while it has quota, with its results unverified. Until then, Codex or Sonnet. Sonnet for tracker checks against main. Haiku for log scans that a person or Sonnet confirms | Haiku alone on checks where a miss matters |
| Coordination sessions | Claude Code on Fable 5.2 for the coordinating session. Planning and design sessions and subagents follow the planning row. Sonnet for every other Claude session and subagent. Codex and Antigravity are unverified for this job | Sonnet for planning or design inside the session |
| First-pass triage | Haiku, and a person or Sonnet confirms its result. Antigravity `gemini-4-argon-high` first while it has quota, unverified | Haiku deciding whether a change is on main from commit subjects alone |

## Matching tiers

Use this table to find the matching reviewer on another provider. Treat a pairing as unverified unless the Support column cites an observation.

| Claude tier | Codex | Antigravity | Support |
| --- | --- | --- | --- |
| Opus 5.5 or Fable 5.2 | `gpt-6.1-sol`, at its highest effort | `gemini-4-argon-high`, which draws down the Antigravity limit the way Fable draws down Claude's | Unverified |
| Sonnet | `gpt-6.1-sol` | `gemini-4-neon-high`, the likely match | [OBSERVATIONS.md](OBSERVATIONS.md) records Codex reviewing Claude's code with success, and Antigravity reviewing scorer and launcher code with mixed results. Everything else is unverified |
| Haiku | None named | None named | Unverified |

## Reviewer by tier

| Review | Codex | Antigravity | Claude fallback | Support |
| --- | --- | --- | --- | --- |
| Tier 2, two runs | `gpt-6.1-sol` at high effort | `gemini-4-argon-high` | Opus 5.5 | Unverified, except Codex review of Claude's code in [OBSERVATIONS.md](OBSERVATIONS.md) |
| Tier 1, one run | `gpt-6.1-sol` at medium effort | `gemini-4-neon-high`, or a Carbon key | Sonnet | Unverified |
| Delta re-review, one run | `gpt-6.1-sol` at medium effort | `gemini-4-neon-high`, or a Carbon key | Sonnet | Unverified |

Claude models share one subscription's quota, and Codex runs on its own subscription. Neither bills per token, and how Antigravity bills is unverified. The Claude Ultra plans have a weekly window and a separate weekly Fable window, and no 5-hour window, as the usage view shows. Antigravity's 5-hour window has been the binding limit for builds, and when it runs low, new Antigravity builds wait and Codex builds. Codex has a weekly window. The Antigravity and Codex window claims come from the owners' reports and are unverified, as is the exact quota for each model.

Antigravity also offers Gemini 4 Carbon (low, medium, and high), Gemini 4 Hydrogen (low through extra high), and two Orchestrator keys. They're available but untested, so their fit is unverified. `gemini-4-argon-high` stays the recommended Antigravity model for review and read-only work until observations say otherwise.

## Routing by quota

Pick the harness by its remaining weekly headroom. Send read-only work, such as reviews, log scans, state checks, and triage, to the harness with the most headroom. Antigravity takes read-only peer work first while it has quota. Keep builds on Codex while it has headroom, and fall back to Sonnet when Codex is out. Every Claude subagent call states its model. That's Sonnet or Haiku, except for planning and design, which run on Opus 5.5 or Fable 5.2.

## Not yet decided

Host ops, console work, grading evaluations, and approver verdicts have no recommendation yet.

## Updating this page

Change a recommendation only when [OBSERVATIONS.md](OBSERVATIONS.md) or a ruling supports it, and say `unverified` for any claim without support. Use [TEMPLATE.md](TEMPLATE.md), check the change against [WRITING.md](WRITING.md), and have it reviewed with [REVIEW.md](REVIEW.md).
