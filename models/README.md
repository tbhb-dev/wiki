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
- Outside the trial below, Codex `gpt-6.1-sol` is the second model family for code review. The reviewer of record checks every finding.
- Review depth follows the tier of the diff, as a starting point. Tier 0 is prose-only changes in the wiki, drafts, research, and spikes repos, and needs only CI. Design PRs under `designs/` aren't tier 0. Tier 1 is most code, tests, refactors, and docs in code repos, and gets one fresh review run. Tier 2 is anything that deletes data or runs destructive commands, touches credentials, rulesets, or permissions, changes agent harness config, deploys, or changes the review and merge path, and gets two review runs. A re-review after a fix covers only the delta, with one run, at any tier. The paths and content a diff touches set the tier, never the maker's judgment. A reviewer may raise a tier, and nothing lowers it.
- Pick the reviewer model by matching tier. Prefer a model of the same tier on a different provider from the build, using the Matching tiers table. When only the build's provider has capacity left, use that provider one tier up: a Sonnet build gets an Opus reviewer. The preceding pairing ruling gives the concrete pairs, and this ruling sets the general choice.
- Trial of Codex builds and Opus reviews. Builds go to Codex `gpt-6.2-sol`, and review runs go to Claude Opus 5.5 at medium effort, because Claude has two Ultra plans of quota against one Codex Pro plan. Builds that Codex can't do stay on Sonnet, with Codex Sol reviewing them. When a Codex review run quotes the provider's refusal of security-test code, a Claude review run on Sonnet replaces it. The trial replaces the Antigravity-first and Argon-reviewer lines. Antigravity takes read-only checks on `gemini-4-hydrogen` once its window resets. The trial holds until one probe run shows the harness accepts `gpt-6.2-sol`, and its fit is unverified until [OBSERVATIONS.md](OBSERVATIONS.md) records results.
- Designs are written on Opus 5.5, and one Codex `gpt-6.2-astra` run reviews each design PR. Other prose-only wiki changes stay tier 0. The Astra model is unverified until the harness accepts it.
- Antigravity runs top-level on the lab hosts. Its runs can't push, because the lab hosts hold no write credential. The owner's session replays a run's commit on its own branch through its own gate, then reviews and commits it.
- Planning and design work, in a session or a subagent, follows the planning ruling: the coordinating session runs on Fable 5.2, and every other planning or design session or subagent runs on Opus 5.5 or Fable 5.2. Every other Claude session and subagent runs on Sonnet, except the Haiku jobs in the Models table.
- Any API spend needs @tbhb's approval first.

## Models

| Job | Models and harnesses | Don't use |
| --- | --- | --- |
| Planning and design | Opus 5.5 or Fable 5.2 at medium effort. One Codex `gpt-6.2-astra` run reviews a design PR | Sonnet or Haiku |
| Building | Codex `gpt-6.2-sol` in the trial, otherwise `gpt-6.1-sol`. Claude builds only where Codex can't, such as work that needs an operating-system sandbox. When Claude builds, it runs on Sonnet | Opus 5.5 or Fable 5.2 for routine edits that a cheaper model handles |
| Code review | Claude Opus 5.5 at medium effort reviews Codex builds in the trial, and Codex Sol reviews builds that stay on Sonnet. A Claude review run on Sonnet replaces Codex when its provider filter refuses the code. The reviewer of record checks every finding and signs | Haiku as the reviewer of record |
| Read-only checks and scans | Antigravity `gemini-4-hydrogen` once its window resets, with its results unverified. Until then, Codex or Sonnet. Sonnet for tracker checks against main. Haiku for log scans that a person or Sonnet confirms | Haiku alone on checks where a miss matters |
| Coordination sessions | Claude Code on Fable 5.2 for the coordinating session. Planning and design sessions and subagents follow the planning row. Sonnet for every other Claude session and subagent. Codex and Antigravity are unverified for this job | Sonnet for planning or design inside the session |
| First-pass triage | Haiku, and a person or Sonnet confirms its result. Antigravity `gemini-4-hydrogen` once its window resets, unverified | Haiku deciding whether a change is on main from commit subjects alone |

## Matching tiers

Use this table to find the matching reviewer on another provider. Treat a pairing as unverified unless the Support column cites an observation.

| Claude tier | Codex | Antigravity | Support |
| --- | --- | --- | --- |
| Opus 5.5 or Fable 5.2 | `gpt-6.2-astra` for the fallback or a hard review, otherwise `gpt-6.1-sol` at high effort | `gemini-4-argon-high`, which draws down the Antigravity limit the way Fable draws down Claude's | Unverified |
| Sonnet | `gpt-6.1-sol` | No match chosen. `gemini-4-hydrogen` is the cheaper Antigravity model | [OBSERVATIONS.md](OBSERVATIONS.md) records Codex reviewing Claude's code with success, and Antigravity reviewing scorer and launcher code with mixed results. Everything else is unverified |
| Haiku | None named | None named | Unverified |

## Reviewer in the trial

| Build | Reviewer | Support |
| --- | --- | --- |
| Codex `gpt-6.2-sol` | Claude Opus 5.5 at medium effort | Unverified |
| Sonnet, where Codex can't build | Codex Sol | [OBSERVATIONS.md](OBSERVATIONS.md) records Codex reviewing Claude's code with success. The Sol model is unverified |
| Either, when the reviewer's provider is out | The build's provider one tier up | Unverified |

Claude models share one subscription's quota, and Codex runs on its own subscription. Neither bills per token, and how Antigravity bills is unverified. The Claude Ultra plans have a weekly window and a separate weekly Fable window, and no 5-hour window, as the usage view shows. Codex has a weekly window. The Antigravity and Codex window claims come from the owners' reports and are unverified, as is the exact quota for each model.

Antigravity keeps two quota pools, one for Gemini models and one for Claude and GPT models. Each pool has a 5-hour meter and a weekly meter. [Google's plans post](https://antigravity.google/blog/changes-to-antigravity-plans) says usage draws down by API price. A cheaper model stretches further, though the post predates Gemini 4. Third-party price lists rank Hydrogen as the cheapest Gemini 4 model, Neon costing more than Argon, and Carbon about level with Argon, and one source disagrees on Carbon. Third parties also report a price rise for Argon in December. If usage follows price, Hydrogen is the cheap reviewer, and high effort adds thinking tokens. These claims come from third parties and stay unverified until OBSERVATIONS.md records measured usage. [Google's model page](https://antigravity.google/docs/models) lists which plans include each model.

Codex shipped GPT-6.2 in two variants, Astra for frontier work and Sol for everyday work. Third-party sources, including [OpenAI's Codex pricing page](https://developers.openai.com/codex/pricing), put Sol at the same API price as `gpt-6.1-sol` and Astra at four times that price. They also give Astra far fewer local messages per 5-hour window than Sol. Sol is the default Codex builder, and Astra is a top-tier model beside Fable, Opus, and Argon, for the fallback or a hard build only. The move from `gpt-6.1-sol` to `gpt-6.2-sol` waits on one probe run that shows the harness accepts it. Until then the move and these figures are unverified.

Antigravity also offers Gemini 4 Carbon (low, medium, and high), Gemini 4 Hydrogen (low through extra high), and two Orchestrator keys. They're available but untested, so their fit is unverified. `gemini-4-hydrogen` is the Antigravity model for read-only work until observations say otherwise.

## Routing by quota

Pick the harness by its remaining weekly headroom. Send read-only work, such as reviews, log scans, state checks, and triage, to the harness with the most headroom. Antigravity takes read-only peer work on Hydrogen once its window resets. Keep builds on Codex while it has headroom, and fall back to Sonnet when Codex is out. Every Claude subagent call states its model. That's Sonnet or Haiku, except for planning and design, which run on Opus 5.5 or Fable 5.2.

## Not yet decided

Host ops, console work, grading evaluations, and approver verdicts have no recommendation yet.

## Updating this page

Change a recommendation only when [OBSERVATIONS.md](OBSERVATIONS.md) or a ruling supports it, and say `unverified` for any claim without support. Use [TEMPLATE.md](TEMPLATE.md), check the change against [WRITING.md](WRITING.md), and have it reviewed with [REVIEW.md](REVIEW.md).
