<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Model observations

This file backs the recommendations in [README.md](README.md). Each line is one finding, grouped by model, and none is a benchmark. Every result comes from a small sample.

## Haiku

- Succeeded at classifying findings and blockers in a short code-review text, and at sorting rows of a tracker file.
- Succeeded at checking JSON run records against field rules, with planted errors only. Subtle anomalies are untested.
- Succeeded at scanning a CI log for cache failures, and said which parts it checked by search.
- Mixed on reading a long CI job log. It found the failing tests and the assertion text, but it misread two file hashes and counted one follow-on error as its own problem. Use it for triage.
- Mixed on checking a draft against a list of private strings. It missed one listed word and flagged two harmless items. Keep a search or a Sonnet read as the check.
- Mixed on reading a host snapshot. It caught a load spike, but it flagged noise and misread a CPU threshold. Treat it as a first-pass filter.
- Mixed on deciding whether a change is on main from commit subjects. It over-called "nothing yet" because work landed under different names. Give it file listings or grep output.

## Sonnet

- Succeeded at a read-only check of tracker rows against the main branch, PR state, and decision files, and marked rows that need host access as uncheckable.
- Succeeded at fixing a script, which then passed its test.

## Codex `gpt-6.1-sol`

- Succeeded at second-family code review, finding blocking security issues in two PRs that the first-family review alone stated less strongly.
- Failed at reviewing a change that tests symlink and patch-export safety. The provider's content filter refused the review four times, so a Claude review run replaced it. The owner of that change reported this.

## Antigravity `gemini-4-argon-high`

- Succeeded at quoting a marker line from `GEMINI.md`, from `AGENTS.md`, and from both files in one directory, in headless one-shot runs. Each file loaded without a flag.
- Succeeded at running a `PreToolUse` hook before each tool call in a session, and at reading a mid-session edit of the hooks file on the next call. The agent never rewrote the file.
- Succeeded at stopping `git push` and `git reset --hard` through a `PreToolUse` hook in headless runs. Exit code 2 and a stdout `decision` of `deny` both worked, and the model saw the reason text. A hook that crashes, times out, or lacks its interpreter is untested.
- Failed at blocking `git push` through the `--settings` permissions key. None of seven shapes tried stopped the push, and the flag rejects unknown keys such as `deny`, `hooks`, and `sandbox`. Rules saved by `/permissions` are untested.
- Mixed on settings keys. `--settings` accepted `statusLine` and `model`, but its schema isn't documented in the help text. Each key needs a probe.
- Failed at writing a design draft from a nested folder of CSV results. Headless runs ended within seconds with no output on all three tries, because headless mode auto-denied the shell command that read the inputs.
- Succeeded at the same draft once the inputs were flat files in one folder. The figures it quoted were accurate against the source results, and one run is the whole test.
- Mixed on labelling evidence in that draft. Its prose was dense, and it labelled some inferred claims as measured, so check each label against the data.

## Antigravity `gemini-4-neon-high`

Each line is the owner's report of one build run, and every run was a single sample. No run could push, so each commit reached the repository as a patch that the owner replayed.

- Succeeded at splitting an oversized prose draft into two files. Every moved line reappeared, and both files came in under the size limit. It dropped ten section anchors without leaving compatibility anchors, and it added a `tags:` field that the sibling appendices lack. No link to the dropped anchors existed.
- Mixed at building a scheduled settings job. It finished in about 15 minutes, and its summary said under 400 code lines. The PR measured 411, partly because of the owner's own additions. It missed home-directory template placeholders in two settings sources, so the job would have written them verbatim, and the owner added a guard.
- Mixed at a narrow config fix. It went beyond the brief and moved an allowlist into the generated top-level config after reading the scanner's source. Its gate passed, and the wider change is under review, so whether it is right is unverified.
- Succeeded at adding a metric gauge to a receiver, with table tests and a catalog row. It finished, the run reported its tests passing, and it followed the brief. The diff isn't reviewed yet.
