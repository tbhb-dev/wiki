---
type: Design Appendix
title: Commit types and scopes checker
description: How the subject checker applies the merged list, its arguments, messages, and exit codes, its failure modes, and its test plan.
tags: [commits, lint, testing, design]
status: draft
parent: README.md
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-opus-5-5
    surface: cli
    at: "2026-11-23T15:44:11Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore pnpm nope -->

# Commit types and scopes checker

Back to [the design](README.md).

Everything here is new and proposed, except where a line says the behavior exists today.

## Arguments

```text
check_subject --repo-root DIR [--base-repo-root DIR | --evaluate | --enforce] [--title] [--list] [SUBJECT | --file PATH]
```

- `--repo-root` must be present. The checker passes it to `load("commits", DIR)` for the scope lists and other settings. Pull request gates supply the head's configuration root.
- `--base-repo-root` supplies the base branch's configuration root. The checker uses the shared loader to read the base mode and applies `enforce` to the title whenever either mode is `enforce`.
- `--title` applies the absolute `limits.title` cap to a pull request title. The shared global file defaults to 64 characters.
- `--evaluate` and `--enforce` override the mode for a standalone run. The reports in plan steps 6 and 7 use `--evaluate`. The checker rejects either override when the caller supplies `--base-repo-root`.
- `--list` prints the merged list, one line per type and per scope with its class and description. An agent reads it before it writes a subject.
- With `--file`, the checker reads the first line of the file, as the commit-msg stage passes it today. With neither a subject nor `--file`, it reads the first line of standard input, as the range target does today.

## The order of checks

| Order | Check | Evaluate | Enforce |
| --- | --- | --- | --- |
| 1 | Apply `limits.title` to all titles, including dependency updates and quoted reverts. | Blocks | Blocks |
| 2 | Pass a commit subject matching the configured quoted `Revert` or `Reapply` form. | Passes | Passes |
| 3 | Check `rewrite_prefixes` and `unfinished` words. Apply `limits.subject` to commit subjects unless a `long_prefixes` entry matches. | Blocks | Blocks |
| 4 | Parse `type(scope)!: summary` with optional scope and `!`. Keep the current scope syntax (`[^()\s][^()]*`) and retain the spelling for the scope checks. | Blocks | Blocks |
| 5 | Check that the parsed type appears in `types`. | Blocks | Blocks |
| 6 | Check the parsed scope against the merged list and the policy for its type. Policy `none` flags any scope. | Advisory | Blocks titles. Advisory for authored commits. |
| 7 | Match `generic` against the normalized subject or parsed summary. | Blocks | Blocks |

In `evaluate` mode the checker prefixes step 6 advisories with `evaluate:`. In `enforce` mode it prefixes authored-commit scope findings with `advisory:`. The planned upgrade makes these findings block in `enforce` mode once the merge queue runs the commit check on `merge_group` as a required check. [Decision D10](decisions.md#d10-which-commit-check-governs-the-merge)

In both modes the shared commit gate blocks unknown types, rewrite prefixes, and empty descriptions. The 72-character subject cap applies with its existing exemptions. Titles use the title cap in place of the subject cap.

## Messages

Each problem is one line. The allow-list messages state the fix.

| Case | Message |
| --- | --- |
| Unknown type | `type "sync" is not allowed; use one of: build, chore, ci, docs, feat, fix, perf, refactor, revert, style, test` |
| Unlisted scope | `scope "e-000001" is not listed; drop the scope, or add it to .tbhb/commits.toml if it names a part of this repository` |
| Near miss | `` scope "`bazel`" is not listed; did you mean "bazel"? `` |
| Wrong class | `scope "designs" is a repository scope, and type "ci" takes only global scopes` |
| Scope on a `none` type | `type "style" takes no scope` |
| Generic summary | `summary "update" is too generic. Say what changed.` |
| Title too long | `title is 65 characters; the limit is 64` |
| Configuration error | `.tbhb/global/commits.toml: missing; run the baseline sync for this repository` |

A near miss is a scope that equals a listed scope after normalizing. Normalizing strips backticks, quotes, and spaces. It then turns `_` into `-` and lowers the case. A scope that a close-match search ranks first, past a fixed cutoff, also counts as a near miss.

## Exit codes

| Exit | When |
| --- | --- |
| 0 | The check passes with no blocking findings, or `--list` succeeds |
| 1 | At least one finding blocks in the effective mode |
| 2 | Usage, configuration, or execution error |

A `notice:` line describes a duplicate entry and leaves the result to the subject checks.

## Range reports

`commit_messages -- [--repo-root DIR] [--base-repo-root DIR | --evaluate] [--pr-number N --title TEXT] BASE HEAD` checks each non-merge commit in `BASE..HEAD`. With a pull request number and title, it checks the title too. The runner defaults to the workspace for local reports and forwards the supplied roots and any `--evaluate` flag to every checker invocation.

The runner records checker output under each commit SHA and records title output under the pull request number. It prints the numbers of commits and titles checked and the finding count for each. Each blocking or advisory finding contributes once. The runner counts `notice:` lines separately and returns the highest checker status. It checks the title even if the commit range is empty and validates configuration even if there is nothing to check.

For a rollout scan, the report walks the default branch's first-parent history in the requested window. For each pull request merge commit, the report reads the merge subject, recovers the title and number from `title (#N)` under [ADR 0020](../../adrs/0020-pull-requests-land-through-merge-commits.md), and invokes the runner with that title and the first and second parents as `BASE` and `HEAD`. It also checks any direct commits in the window. All checks use the configuration from the scanned default branch tip.

The rollout report retains each repository's commit and title results and sums both finding counts. A range enumeration, title extraction, configuration, or execution error makes the report incomplete. To advance in step 7, every repository must complete the seven-day scan with zero commit findings and zero title findings, including advisories.

## Callers

| Caller | Invocation | Stage |
| --- | --- | --- |
| Shared CI workflow | `@tbhb_rules//lint:commit_messages -- --repo-root "$PR_CONFIG_ROOT" --base-repo-root "$BASE_CONFIG_ROOT" --pr-number "$PR_NUMBER" --title="$PR_TITLE" "$BASE_SHA" "$PR_HEAD_SHA"`, with the title in an environment variable | Each pull request run |
| `tbhbdev alpha pr check` and `pr create` | `bazelisk run @tbhb_rules//lint:commit_subject -- --repo-root <worktree> --title -- <title>` | Before opening. The checker supplies the title length check from `limits.title`. |
| `tbhbdev alpha merge` | `commit_subject -- --repo-root <head-config> --base-repo-root <live-base-config> --title -- <latest-title>` | Before merging, alongside validation of the final subject with `limits.subject` |
| A future tools commit command | `commit_subject` without `--title` | Before a commit or a push |
| The `commit-msg` hook | `commit_subject` through a `bazel_env` shim, with `--file` | At commit creation, before push. A later change after title enforcement. |

Both Bazel targets bundle `check_subject.py` with the loader from the pinned tools source, `tbhb_rules_tbhbdev_src`, and the pinned `cue` binary. CI, `pr check`, and `pr create` all apply the same `limits.title` key through that loader and checker. `pr check` and `pr create` remove their hard-coded title cap and keep their other pull request checks.

### Pull request configuration

CI reads the head and base SHAs from `github.event.pull_request.head.sha` and `github.event.pull_request.base.sha`. The merge command reads the current title, `headRefOid`, and `baseRefName` together from the pull request, then fetches `refs/heads/<baseRefName>` from the target repository for the live base SHA and mode. Both callers resolve the exact head SHA, fetching the pull request head ref if the object is missing and verifying that the requested SHA is available.

Both callers copy the tracked files `.tbhb/global/commits.toml`, `.tbhb/schema/commits.cue`, and the optional `.tbhb/commits.toml` from the head tree into a temporary directory for `--repo-root`. They copy only the global file and schema from the base tree into a second directory for `--base-repo-root`. The exports preserve relative paths and accept regular file blobs. The checked-out tree serves as the Bazel workspace. The callers remove both exports after the checks.

The loaded head configuration supplies the scope lists and the other checker settings. The base configuration supplies only its mode. Both callers apply the stricter mode to the title. CI also checks the authored commits in `BASE..HEAD` from the event, with advisory scope findings in both modes.

CI reruns use the base from the original event. The merge command checks the latest title in the fetched base mode if that mode is stricter. A blocking title finding or a check error stops the merge.

The merge request requires the hash of the checked head. A head change requires a fresh check.

## Failure modes

| Failure | Behavior |
| --- | --- |
| `.tbhb/global/commits.toml` is missing | The loader reports a configuration error with the sync hint. |
| The global file fails the schema | The loader reports each `cue vet` finding. |
| `.tbhb/schema/commits.cue` is missing | The loader reports the missing schema as a configuration error. |
| The pinned `cue` binary is missing | The loader reports the missing binary as a configuration error. |
| `.tbhb/commits.toml` is absent | The global list alone applies. |
| `.tbhb/commits.toml` is unreadable, isn't TOML, or fails `#Repo` | The loader reports a configuration error with the file and the finding. |
| The repository file repeats a global scope | The loader keeps the global entry, and the checker prints a `notice:` line and continues checking the subject. |
| The checker's own code raises an unexpected error | The gate reports an execution error and fails closed. |
| The subject is empty | The checker reports a blocking form finding in both modes. |
| A pull request head predates the enforce sync | If the base says `enforce`, CI and the merge command enforce the title's scope rules using the head's scope lists. |
| CI or the merge command cannot read the base revision, schema, or mode | The check fails with a configuration error. |

## Test plan

The rules maintainer writes the checker tests under `lint/private/commit_messages/`, and the tools maintainer writes the loader tests in tools. Each fixture is a small repository root with its own `.tbhb/` files.

### Loader

- A valid global file and no repository file: the merged values equal the global values, and every scope's class is `global`.
- A repository file that adds a scope: the scope appears with class `repo`.
- A repository file that repeats a global scope: the result keeps one entry of class `global` and adds a notice.
- A missing global file, a global file with a TOML error, a global file that fails `#Global`, and a missing schema: each raises `ConfigError` naming the file.
- A repository file that's a directory, isn't readable, has a TOML error, or sets `mode`, `limits`, `types`, or `subjects`: each raises `ConfigError`.
- A scope name that breaks the grammar or runs past 24 characters, in either file: `ConfigError`.
- A missing `schema_version` or `schema_version = 2` in either file raises `ConfigError`.
- Control and separator characters in a scope name or description raise `ConfigError`. Run the [schema verification cases](C-configuration.md#schema-verification) with each configuration definition. A one-line description passes.
- A global title limit above the subject limit raises `ConfigError`. A title limit of 65 with a subject limit of 72 passes.
- A section name with `/` or `..`: the loader rejects it before it reads any path.
- The same fixture driven through `load("taxonomy", …)` with a schema laid out for the taxonomy, to prove the loader doesn't contain code specific to commits.

### Checker

- **Each type.** For all 11 types, a subject with no scope passes in enforce mode.
- **Each scope class.** For each type, test a global scope and a repository scope as titles and as authored commits in both modes. Each policy finding blocks a title in `enforce` mode and produces an advisory for a title in `evaluate` mode or an authored commit in either mode. A test fixture sets one type to `none` and one to `repo` to cover the two policies the seed omits.
- **Unknown scope.** As a title, `feat(nope): add x` exits 1 in `enforce` mode and prints an `evaluate:` line with exit 0 in `evaluate` mode. As an authored commit it prints an advisory and exits 0 in both modes.
- **Unknown type.** `sync: copy the shared files` fails with the type message in both modes.
- **Spellings.** `_`, `` `bazel` ``, `claude_code`, `src/main.py`, `a,b`, and `e-000001` each produce a scope finding. The first three get the near-miss text where a listed scope matches.
- **Breaking marker.** `feat(mcp)!: require a relay endpoint` passes, and `!` counts toward the length.
- **Lengths.** With the default limits, a 64-character title passes and a 65-character title fails in both modes. An ordinary 72-character commit subject passes. Adding one character makes it fail with exit 1 in both modes. A 90-character Renovate `chore(deps): update …` subject passes as a commit and fails as a title in both modes. An over-limit quoted revert title also fails in both modes.
- **Shared title limit.** Set `limits.title = 65` with `limits.subject = 72`. Verify that CI, `pr check`, and `pr create` all accept the same valid 65-character title and reject it after adding one character.
- **Final subject limit.** The merge command validates the final subject against `limits.subject`, so a 65-character title plus a space and `(#9999)` fails at merge with a 73-character subject. With a 72-character subject cap, set `limits.title` to 63 to fit every pull request number up to five digits.
- **Rewrite prefix.** `fixup! feat: add export` fails with exit 1 in both modes and reports a rewrite-prefix finding.
- **Summary quality.** `feat: update` fails with exit 1 and a generic-summary finding in both modes. Check it as a commit subject and as a title through each step 5 caller.
- **Existing behavior.** Run the cases in today's `test_check_subject.py` in both modes with the seed global file. Preserve existing blocking commit results and exemptions. Use exit 2 for usage and configuration errors.
- **Duplicate global scope.** With `deps` repeated in the repository file, `fix(deps): repair the lockfile` prints a notice and passes. In both modes, `fix(deps): update` prints the same notice and fails the summary check.
- **Missing global file.** Exit 2 with the sync hint, in both modes.
- **Range target.** A range with one good commit and one commit with an unlisted scope exits 0 in both modes. It prints the advisory under the hash of the second commit and reports two checked commits and one finding. The rollout report counts the advisory as a finding.
- **Range failures.** A range containing an authored `fixup!` commit exits 1 in both modes. A missing global file causes exit 2 and an incomplete report even for an empty range.
- **Rollout title finding.** Create a pull request fixture with title `feat(nope): add export` and one authored commit `feat: add export`. Merge it with subject `feat(nope): add export (#42)`. In evaluate mode the runner checks one commit and one title, exits 0, and records one advisory under pull request 42 and zero commit findings. The seven-day scan recovers the title from the merge subject and blocks step 7.
- **CI title caller.** Use an enforce-mode fixture where the pull request head adds a scope and its title uses it. Leave the scope out of the checked-out test merge tree and verify that CI still passes the title using the event's head SHA.
- **Merge title caller.** Use the same pull request fixture from a worktree without the added scope. Verify that the merge command checks the title using the API's head SHA and sends that SHA as the merge precondition. If the head changes, the merge fails until the command checks the new revision.
- **Title mode at the two revisions.** Set the pull request head to `evaluate` and the base and checked-out merge tree to `enforce`. Put a repository scope only in the head and a global scope only in the base. Verify that CI and the merge command accept a title with the head-only scope and reject one with the base-only scope. Reverse the modes and verify the same results. With both modes set to `evaluate`, the base-only scope yields an advisory.
- **Base configuration errors.** An unavailable base revision or invalid base mode blocks CI and the merge command.
- **Base advances after CI.** Start in `evaluate` with an unlisted scope in the title. Advance the base to `enforce` and leave the head unchanged. Verify that CI still passes on a rerun of the original event and the merge command blocks on the latest title.
- **The `commit-msg` hook.** In both modes, a commit with an unlisted scope or a scope outside its type's policy produces an advisory before push. A `fixup!` subject or empty description still blocks the commit.
- **Mutation testing.** The checker and the loader run under the repositories' mutation test setup, and every surviving mutant gets a test or a recorded reason.
