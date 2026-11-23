---
type: Design Appendix
title: Commit types and scopes checker
description: How the subject checker enforces the merged list, its arguments, messages, and exit codes, its failure modes, and its test plan.
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
check_subject --repo-root DIR [--title] [--evaluate | --enforce] [--list] [SUBJECT | --file PATH]
```

- `--repo-root` must be present. The checker passes it to `load("commits", DIR)` and reads nothing else from disk except `--file`.
- `--title` applies the absolute `limits.title` cap to a pull request title.
- `--evaluate` and `--enforce` override the global file's `mode` for one run. The reports in plan steps 6 and 7 use `--evaluate`, and no caller in a gate passes either flag.
- `--list` prints the merged list, one line per type and per scope with its class and description. An agent reads it before it writes a subject.
- With `--file`, the checker reads the first line of the file, as the commit-msg stage passes it today. With neither a subject nor `--file`, it reads the first line of standard input, as the range target does today.

## The order of checks

1. Check `limits.title` for every title, including titles that match a `long_prefixes` or `passthrough` entry. An over-limit title blocks in both modes.
2. Pass a commit subject that starts with a `passthrough` word, a space, and a quoted subject.
3. Report a policy finding for each `rewrite_prefixes` start, `unfinished` word, or commit subject that exceeds `limits.subject`. A matching `long_prefixes` entry exempts a commit subject from the length check alone.
4. Parse `type(scope)!: summary`, with optional scope and `!`. Report a form finding if parsing fails. The parser accepts any characters inside the parentheses so the scope check can show the original spelling.
5. Report a type finding if the parsed type is absent from `types`.
6. For a parsed scope, report a finding if the scope is absent from the merged list or if the listed type's policy rejects its class. Policy `none` rejects every scope.
7. Report a summary finding if the normalized summary matches a `generic` entry.

Evaluate mode prefixes each policy finding from steps 3 through 7 with `evaluate:` and treats it as advisory. Enforce mode treats those findings as failures. The summary check already exists for commit subjects. Applying it to titles in step 5 of the rollout adds a finding without adding a failure during evaluation.

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
| 1 | The title exceeds its limit, or enforce mode has at least one policy finding |
| 2 | Usage, configuration, or execution error |

A `notice:` line describes a duplicate entry and leaves the result to the subject checks.

## Range reports

`commit_messages -- [--repo-root DIR] [--evaluate] BASE HEAD` checks each non-merge commit in `BASE..HEAD`. The runner defaults to the workspace for local reports and forwards the configuration root and any `--evaluate` flag to every checker invocation. Pull request CI supplies the root from the pull request head and uses the configured mode.

The runner captures the checker's standard output and error output under each commit SHA, including output from successful checks. It prints the number of checked commits and counts each policy finding once, including each `evaluate:` line. It counts `notice:` lines separately. The range returns the highest checker status. For an empty range, the runner validates the configuration before reporting zero counts. A range enumeration, configuration, or execution error makes the report incomplete.

The rollout report retains each repository's range output and sums the policy finding counts across repositories. The rules maintainer requires a completed scan for every repository and zero findings for the seven-day report to qualify for step 7.

## Callers

| Caller | Invocation | Stage |
| --- | --- | --- |
| Shared CI workflow, per commit | `@tbhb_rules//lint:commit_messages -- --repo-root "$PR_CONFIG_ROOT" "$BASE_SHA" "$PR_HEAD_SHA"` | Each pull request run |
| Shared CI workflow, title | `@tbhb_rules//lint:commit_subject -- --repo-root "$PR_CONFIG_ROOT" --title -- "$PR_TITLE"`, with the title in an environment variable | Each pull request run |
| `tbhbdev alpha pr check` and `pr create` | `bazelisk run @tbhb_rules//lint:commit_subject -- --repo-root <worktree> --title -- <title>` | Before opening, alongside the existing title checks |
| `tbhbdev alpha merge` | The same title target as CI, with the latest title and the root for that pull request head | Before merging, alongside validation of the final subject and its 72-character cap |
| A future tools commit command | The same, without `--title` | Before a commit or a push |
| The commit-msg hook | The same target, through a `bazel_env` shim, with `--file` | A later change, after enforcement |

Both Bazel targets bundle `check_subject.py` with the loader from the pinned tools source, `tbhb_rules_tbhbdev_src`, and the pinned `cue` binary. A local run and CI use the same checker code at the same rules pin.

### Pull request configuration

CI reads the head SHA from `github.event.pull_request.head.sha`. The merge command reads the current title and `headRefOid` together from the pull request. Both resolve that exact SHA in the target repository, fetching the pull request head ref if the object is missing and verifying that the fetch gives access to the requested SHA.

Both callers copy the tracked files `.tbhb/global/commits.toml`, `.tbhb/schema/commits.cue`, and the optional `.tbhb/commits.toml` from that commit tree into a temporary directory with the same relative paths. They accept regular file blobs and leave the optional file absent if the tree omits it. They pass the directory as `--repo-root` and remove it after the checks. The checked-out tree serves as the Bazel workspace and the exported blobs supply the checker's configuration. If the caller can't resolve the revision or read its configuration, the check fails. The merge request includes the checked head SHA as a precondition, so a head change requires a fresh check.

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
| The subject is empty | The checker reports a form finding under the selected mode. |

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
- A scope description with an embedded newline raises `ConfigError`. A one-line description passes.
- A section name with `/` or `..`: the loader rejects it before it reads any path.
- The same fixture driven through `load("taxonomy", …)` with a schema laid out for the taxonomy, to prove the loader doesn't contain code specific to commits.

### Checker

- **Each type.** For all 11 types, a subject with no scope passes in enforce mode.
- **Each scope class.** For each type, a subject with a global scope and one with a repository scope pass or fail exactly as the type's policy says. A test fixture sets one type to `none` and one to `repo` to cover the two policies the seed doesn't use.
- **Unknown scope.** `feat(nope): add x` fails in enforce mode with exit 1, and prints an `evaluate:` line with exit 0 in `evaluate` mode.
- **Unknown type.** `sync: copy the shared files` fails with the type message.
- **Spellings.** `_`, `` `bazel` ``, `claude_code`, `src/main.py`, `a,b`, and `e-000001` each fail, and the first three get the near-miss text where a listed scope matches.
- **Breaking marker.** `feat(mcp)!: require a relay endpoint` passes, and `!` counts toward the length.
- **Lengths.** A 64-character title passes with `--title` and a 65-character title fails in both modes. A 72-character commit subject passes and a 73-character subject fails in enforce mode. A 90-character Renovate `chore(deps): update …` subject passes without `--title` and fails with it in both modes. An over-limit quoted revert title also fails in both modes.
- **Summary quality.** Pinned `cog verify` accepts `feat: update`. With `--title`, the checker emits an `evaluate:` summary finding and exits 0 in evaluate mode, then exits 1 on the same title in enforce mode. Run the same assertions on a commit subject and through the step 5 title callers.
- **Existing behavior.** Run the cases in today's `test_check_subject.py` in enforce mode with the seed global file. Preserve the established commit results except for the scope policy and the usage and configuration status defined here.
- **Duplicate global scope.** With `deps` repeated in the repository file, `fix(deps): repair the lockfile` prints a notice and passes. In enforce mode, `fix(deps): update` prints the same notice and fails the summary check.
- **Missing global file.** Exit 2 with the sync hint, in both modes.
- **Range target.** With the global file set to enforce, a range with one good commit and one unlisted scope fails with exit 1. The same range with `--evaluate` exits 0, prints the advisory under the second commit's SHA, and reports two checked commits and one finding. The rollout report treats that scan as having a finding. A range with a missing global file exits 2 and yields an incomplete report even if the range is empty.
- **CI title caller.** Use an enforce-mode fixture where the pull request head adds a scope and its title uses it. Leave the scope out of the checked-out test merge tree and verify that CI still passes the title using the event's head SHA.
- **Merge title caller.** Use the same pull request fixture from a worktree without the added scope. Verify that the merge command checks the title using the API's head SHA and sends that SHA as the merge precondition. If the head changes, the merge fails until the command checks the new revision.
- **Mutation testing.** The checker and the loader run under the repositories' mutation test setup, and every surviving mutant gets a test or a recorded reason.
