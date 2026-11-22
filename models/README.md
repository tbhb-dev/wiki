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
- The review pair follows the model that wrote the diff. A diff from a Codex run, or written by the owner's own session, gets Codex and Antigravity as full peers. A diff with any part from an Antigravity run, Neon or Argon, gets Codex and a Claude review run, and an Antigravity review doesn't count. Codex's provider filter refuses some security-test code. When a Codex review run quotes that refusal, a Claude review run replaces the Codex half.
- Antigravity runs top-level on the lab hosts. Its runs can't push, because the lab hosts hold no write credential. The owner's session replays a run's commit on its own branch through its own gate, then reviews and commits it.
- During the Antigravity trial, new design and build work goes to Antigravity first, then Codex. Jobs that need shell commands or research go to Codex for now, because headless Antigravity auto-denies shell commands. Claude takes only coordination, review, and work that neither can do. The trial overrides the planning-and-design and Codex-builds lines preceding it. Each owner records the quality of a task in [OBSERVATIONS.md](OBSERVATIONS.md).
- Some builds go to Antigravity `gemini-4-neon-high`, and hard builds to the same model at extra-high effort. Codex stays a builder too. Neon's observations are in [OBSERVATIONS.md](OBSERVATIONS.md), and how well it builds is unverified.
- Only the coordinating session runs on Fable 5.2. Design sessions run on Opus 5.5. Every other session and every subagent runs on Sonnet. Sonnet is also the default model for new Claude Code sessions, as a backstop when a launch names no model.
- Any API spend needs @tbhb's approval first.

## Models

| Job | Models and harnesses | Don't use |
| --- | --- | --- |
| Planning and design | Opus 5.5 or Fable 5.2 at medium effort | Sonnet or Haiku |
| Building | Codex `gpt-6.1-sol` by default. Claude builds only where Codex can't, such as work that needs an operating-system sandbox. When Claude builds, it runs on Sonnet | Opus 5.5 or Fable 5.2 for routine edits that a cheaper model handles |
| Building, Antigravity share | `gemini-4-neon-high`, or the same model at extra-high effort for a hard build, for a share of builds. Its commit comes back as a patch that the owner replays and reviews. Its fit is unverified | An Antigravity review of a diff that Antigravity built |
| Code review | Codex `gpt-6.1-sol` and Antigravity `gemini-4-argon-high` as full peers for the review read. A diff built by Antigravity gets Codex and Claude. A Claude review run replaces Codex when its provider filter refuses the code. How well Antigravity reviews is unverified. The reviewer of record checks every finding and signs | Haiku as the reviewer of record |
| Read-only checks and scans | Antigravity `gemini-4-argon-high` first while it has quota, with its results unverified. Until then, Codex or Sonnet. Sonnet for tracker checks against main. Haiku for log scans that a person or Sonnet confirms | Haiku alone on checks where a miss matters |
| Coordination sessions | Claude Code on Fable 5.2 for the coordinating session only, and Opus 5.5 for design sessions. Sonnet for every other session and every subagent. Codex and Antigravity are unverified for this job | Sonnet for planning or design inside the session |
| First-pass triage | Haiku, and a person or Sonnet confirms its result. Antigravity `gemini-4-argon-high` first while it has quota, unverified | Haiku deciding whether a change is on main from commit subjects alone |

Claude models share one subscription's quota, and Codex runs on its own subscription. Neither bills per token, and how Antigravity bills is unverified. The Claude Ultra plans have a weekly window and a separate weekly Fable window, and no 5-hour window. Antigravity's 5-hour window has been the binding limit for builds, and when it runs low, new Antigravity builds wait and Codex builds. Codex has a weekly window. No one has verified the exact quota for each model.

Antigravity also offers Gemini 4 Carbon (low, medium, and high), Gemini 4 Hydrogen (low through extra high), and two Orchestrator keys. They're available but untested, so their fit is unverified. `gemini-4-argon-high` stays the recommended Antigravity model until observations say otherwise.

## Routing by quota

Pick the harness by its remaining weekly headroom. Send read-only work, such as reviews, log scans, state checks, and triage, to the harness with the most headroom. Antigravity takes read-only peer work first while it has quota. Keep builds on Codex while it has headroom, and fall back to Sonnet when Codex is out. Every Claude subagent call states its model. That's Sonnet or Haiku, except for planning, which runs on Opus 5.5 or Fable 5.2.

## Not yet decided

Host ops, console work, grading evaluations, and approver verdicts have no recommendation yet.

## Updating this page

Change a recommendation only when [OBSERVATIONS.md](OBSERVATIONS.md) or a ruling supports it, and say `unverified` for any claim without support. Use [TEMPLATE.md](TEMPLATE.md), check the change against [WRITING.md](WRITING.md), and have it reviewed with [REVIEW.md](REVIEW.md).
