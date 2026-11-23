---
type: Design Draft
title: Commit types and scopes
description: Shared commit types and scopes, with scope enforcement on pull request titles and scope advisories on authored commits.
tags: [commits, conventional-commits, configuration, rules, design]
status: draft
owner: Rules maintainer
updated: "2026-11-23"
decisions_open: 0
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-opus-5-5
    surface: cli
    at: "2026-11-23T15:44:11Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Commit types and scopes

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Rules maintainer | 2026-11-23 | Nothing |

## Summary

Commit subjects across the organization use 11 fixed types and any scope a writer invents. Over the last 60 days, history gained 425 distinct repository-and-scope pairs, and agents keep adding more. Readers can no longer tell from a scope what changed. On 2026-11-23 the operator decided that types and scopes come from an allow list.

This design moves the types and scopes into a global file, with repository-specific scopes in `.tbhb/commits.toml`. The tools loader merges the two files, and `check_subject` starts in `evaluate` mode. When `enforce` begins, scope findings block pull request titles and remain advisory for authored commits.

The ruleset requires a passing title check through continuous integration (CI), including for GitHub auto-merge. The merge command rechecks the title for early warning.

## Proposal

The built system has five parts, and git stores their state.

- **The global file**, `.tbhb/global/commits.toml` in each repository, lists the types, the scope policy for each type, the global scopes, the length limits, the subject word lists, and the rollout mode. The rules maintainer, the role that maintains tbhb-dev/rules, edits its source in the rules baseline. The baseline sync copies it byte for byte into each repository.
- **The repository file**, `.tbhb/commits.toml`, lists the scopes for the repository's own parts. The role that maintains the repository edits it, and the baseline sync leaves it alone.
- **The schema**, `.tbhb/schema/commits.cue`, defines both files in the Configure Unify Execute (CUE) language. The rules maintainer edits it, and the baseline copies it the same way as the global file.
- **The loader**, `load(section, repo_root)` in tbhb-dev/tools, validates the global file and then the repository file against the schema and merges them. The tools maintainer, the role that maintains tbhb-dev/tools, writes it. The issue taxonomy design calls the same loader with `section = "taxonomy"`.
- **The checker**, `check_subject` in tbhb-dev/rules, checks a subject or a title against the merged list. CI checks each authored commit and the pull request title. Tools' pull request commands check the title before they open or merge a pull request.

A *global scope* refers to something that the shared baseline or a rules gate puts in each repository, or in each repository of a kind, with one meaning across repositories. Each *repository scope* refers to a part of one repository, such as a package, a directory, or another repository it writes about. The *merged list* is the union of the two, and each entry keeps its class.

## How it works

### Scope classes

A scope is global only when it passes three tests:

1. The baseline, a rules gate, or the shared Renovate configuration provides the thing it refers to.
2. It means the same thing wherever it appears.
3. It isn't the name of a repository.

Use in three or more repositories prompts a review against these tests, and the tests decide. A repository name stays a repository scope for two reasons. The global file is public, and a private repository's name must stay out of it, and `docs(tools)` means one thing in a design repository and another in tools itself. [C. Configuration](C-configuration.md#the-global-seed) lists the 16 seed scopes and the reason for leaving out each other candidate.

### The policy for each type

Each type in the global file has one scope policy. `none` rejects any scope, `global` accepts global scopes, `repo` accepts repository scopes, and `any` accepts any scope on the merged list. A missing scope passes for every type.

The operator confirmed the table below in [decision D2](decisions.md#d2-adopt-the-scope-policy-for-each-type).

| Type | Policy | Subjects with no scope in 60 days |
| --- | --- | --- |
| `build` | `global` | 146 of 180 |
| `ci` | `global` | 32 of 70 |
| `chore` | `any` | 297 of 1,186 |
| `docs` | `any` | 216 of 601 |
| `feat` | `any` | 158 of 727 |
| `fix` | `any` | 221 of 779 |
| `perf` | `any` | 0 of 2 |
| `refactor` | `any` | 9 of 34 |
| `revert` | `any` | 0 of 1 |
| `style` | `any` | none used |
| `test` | `any` | 32 of 116 |

`build` and `ci` change the shared toolchain and workflows. In the spot check, 33 of 42 scoped `build` and `ci` subjects used a seed scope, one more used a backtick-wrapped seed scope, and the other 8 referred to one repository's own job and read as well with no scope. `feat` and `fix` keep global scopes because Renovate writes `fix(deps)` and repositories add features to their own Model Context Protocol (MCP) servers and agent files. 149 scoped `feat` and `fix` subjects in the spot check used a seed scope. [A. Evidence](A-evidence.md#subjects-on-main) gives the counts, and [B. Alternatives](B-alternatives.md) gives the rejected policies.

### The files

| File | Source in tbhb-dev/rules | Written by | Repository may change it |
| --- | --- | --- | --- |
| `.tbhb/global/commits.toml` | `repo/baseline/files/tbhb/global/commits.toml` | Baseline sync, mode `bytes` | No |
| `.tbhb/schema/commits.cue` | `repo/baseline/files/tbhb/schema/commits.cue` | Baseline sync, mode `bytes` | No |
| `.tbhb/commits.toml` | none | The repository | Yes, scopes only |

The brief that commissioned this design proposed `config/global/commits.toml` and `config/schema/commits.cue` in rules, and this design moves them. The schema of the baseline manifest rejects a source path outside `repo/baseline/`, and the baseline stores shared files under `files/` without the leading dot. The loader reads only files in the worktree, and the baseline sync puts the copies there, as it does today for the file of pull request sections. [Decision D4](decisions.md#d4-where-do-the-files-live) records the change and the paths for the taxonomy section.

### Loading and merging

1. The caller passes a section name and a repository root.
2. The loader reads `.tbhb/global/<section>.toml`. Any missing, unreadable, or malformed file raises an error.
3. The pinned `cue` binary vets the file against `#Global` in `.tbhb/schema/<section>.cue`. Schema findings and a missing binary raise errors.
4. Next the loader reads `.tbhb/<section>.toml` if the file exists, and uses the global values alone if it doesn't. Read and parse failures raise errors.
5. `cue vet` checks the repository file against `#Repo`. `#Repo` declares only the extendable tables, and a repository file that sets a type, a policy, a limit, or the mode fails the schema.
6. Extendable tables merge by key, global entries first. When a repository key repeats a global key, the loader keeps the global entry and adds a notice. Adding a global scope later can't break a repository that listed it first.
7. The loader returns the merged values, the class of each entry, and the notices. Error messages give the file and the finding.

Repository values extend the global list and can't narrow it. The global file stores every value list: types, scopes, limits, generic summaries, rewrite prefixes, and the dependency-update exemptions. The checker's code keeps only the logic. [C. Configuration](C-configuration.md) gives both files, the schema, and the loader contract.

### Checking a subject

`check_subject` parses the subject as `type(scope)!: summary`, then checks it against the merged list. It reports one problem per line, and for a near miss such as a backtick-wrapped or `snake_case` spelling, it suggests the nearest listed scope. It reads the limits and word lists from the global file. A duplicate scope adds a notice and leaves the subject checks in control of the result.

Tools' `pr check` and `pr create` call `bazelisk run @tbhb_rules//lint:commit_subject -- --repo-root <root> --title -- <title>`, a new target over the checker. The merge command uses the same title target with the live base mode, and CI uses the range target, `@tbhb_rules//lint:commit_messages`, to check authored commits and the title. Both targets use the loader from the tools source that rules already pins for its review policy test. [D. Checker](D-checker.md) gives the messages, arguments, exit codes, and test plan.

### Pull request titles

A pull request title becomes the first line of the merge commit with a space and a `(#N)` suffix. The checker applies `limits.title` to every title, including Renovate titles, and defaults to 64 characters. Every type accepts the breaking marker `!` before the colon, as in `feat(mcp)!: require a relay endpoint`.

The merge command validates the final subject against `limits.subject`, so a 65-character title plus a space and `(#9999)` fails at merge with a 73-character subject. With a 72-character subject cap, set `limits.title` to 63 to fit every pull request number up to five digits.

Shared CI and tools' `pr check` and `pr create` use `limits.title` through the same loader and checker. For the title, CI and `tbhbdev alpha merge` load the scope lists from the exact head of the pull request and use the stricter of the head and base modes. [D7](decisions.md#d7-which-commits-configuration-does-a-pull-request-check-read)

CI checks the title on `pull_request` events `opened`, `edited`, `synchronize`, and `reopened`. The ruleset requires `conclusion`, which fails unless CI reports a successful title check. This blocks the GitHub merge button and auto-merge on failed, skipped, or missing title results. [D. Checker](D-checker.md#required-title-check)

The merge command fetches the live base and rechecks the latest title for early warning. If the title passes, the command appends a space and `(#N)` to form the merge subject and requires the hash of the checked head in the merge request.

### Rollout

The global file starts in `evaluate` mode, with scope advisories for commits and titles. In `enforce` mode, the checker blocks titles with scope findings and still reports advisories for authored commits.

The planned `commit-msg` hook flags scope findings when authors create commits, before they push. A merge can still include a commit with an unlisted scope. Authored-commit scope findings become blocking in `enforce` mode once the merge queue runs the commit check on `merge_group` as a required check. [B. Alternatives](B-alternatives.md#where-to-enforce-scopes)

Each repository's maintainer scans the last 60 days with `--evaluate` and adds the repository scopes that pass the class tests. The scan covers authored commits and pull request titles from merge-commit subjects. The report counts both blocking findings and advisories.

Before a repository's base flips to `enforce`, the operator checks the titles of open pull requests with the proposed base mode and the scope lists from each head. The operator lists the failing pull requests and fixes the scope configuration or changes the titles before the flip. Once every repository completes the seven-day scan with zero findings and passes the check of open titles, the rules maintainer sets `mode = "enforce"` and the sync copies the change out.

To roll back title scope enforcement, sync `mode = "evaluate"` to the base and update open heads that still declare `enforce`. Those heads keep blocking title scope findings until both revisions declare `evaluate`.

Nobody rewrites history. The checks read only the commits of an open pull request and its title, and the reports read history without changing it. `git revert` of an old commit writes `Revert "<subject>"`, which passes unchanged.

## Risks and unknowns

| Risk or unknown | Effect if it goes wrong | What handles it | Evidence |
| --- | --- | --- | --- |
| A repository bumps its rules pin before the sync copies the global file | Each check in that repository fails with a configuration error | Step 2 copies the file into each repository before step 4 reads it, and the error message gives the missing file and the sync step | New |
| Agents now invent repository scopes in `.tbhb/commits.toml` | The list grows as fast as scopes did | Each addition is a reviewed diff with a description, and the checker's message states the class tests | New |
| The issue taxonomy design defines a different loader contract | Two loaders, or a rework | Appendix C defines the contract for any section, and whichever design merges second adopts the first one's contract | Unverified: that design isn't written yet |
| The installed tools version differs from the tools source pinned in rules | The local check and CI disagree | Both run through the same Bazel target, which uses the pinned source | Existing |
| The seeded CI workflow leaves out `edited` | CI can show a pass for an old title | Step 5 adds `edited` and requires a successful title check through `conclusion` | Existing: the seeded CI workflow listens to the default `pull_request` events |
| The pull request head changes between the checks and the merge | The checks used an older revision and its configuration | The merge request requires the checked head SHA, so the operator retries the checks on the new revision | New |
| The base flips to `enforce` after CI passes in `evaluate` | GitHub can accept a passing result on a head behind the base. The merge command can also use the old mode if it fetches the base before the flip | The operator checks the titles of open pull requests before the flip. In option B, `merge_group` checks the base that GitHub uses for the merge | New. Accepted `TECH-DEBT` during the mode flip |
| An authored commit has an unlisted scope | A merge can include the commit | The `commit-msg` hook flags the scope before push. The required check in the merge queue will block commits with scope findings in `enforce` mode | New |
| The seed list misses a common shared scope | Reports in `evaluate` mode fill with one value | The seven-day report shows it, and the rules maintainer adds it under D3 | New |
| One repository's automation writes the `sync` type | Its commits fail after enforcement | That automation moves to `chore(sync)`, and the repository lists `sync` | Existing: 11 subjects |

## Plan

| Step | What lands | Done when | Owner |
| --- | --- | --- | --- |
| 1 | The schema and the global file in the rules baseline, with `mode = "evaluate"`, manifest entries, and `.tbhb/*.toml` in `never_write` | `bazelisk test //repo/baseline:manifest_test` passes in rules and `cue vet` accepts the seed file | Rules maintainer |
| 2 | Baseline sync pull requests that copy both files into each repository, rules included | Each repository's main has `.tbhb/global/commits.toml` byte-equal to the rules source | Rules maintainer |
| 3 | `load(section, repo_root)` in tools with the contract in appendix C | The loader tests in [D. Checker](D-checker.md#test-plan) pass under mutation testing | Tools maintainer |
| 4 | `check_subject` reads the merged list, the `commit_subject` target, and the rules pin of that tools source | The rules test plan in appendix D passes | Rules maintainer |
| 5 | Tools and CI check titles. Update the seed and existing workflows to run the required title check. The merge command rechecks the title for early warning. | An unlisted scope in an authored commit remains advisory in CI in both modes. The merge command blocks a title with an unlisted scope if the base flips to `enforce` before the fetch, even with earlier passing CI. `feat: update` and a title beyond the configured limit both fail. | Tools maintainer, rules maintainer |
| 5a | At checker release, the administrator account applies the ruleset that requires `conclusion` with the title check as a required input. Step 5 depends on this change. | CI checks titles on `opened`, `edited`, `synchronize`, and `reopened`. Failed, skipped, and missing title results fail `conclusion` and block the GitHub merge button and auto-merge. | Administrator account |
| 6 | Each repository's `.tbhb/commits.toml`, seeded from its 60-day report in `evaluate` mode | Each maintainer reviews the range report's recorded findings and uses its scope counts to seed the repository list. | Each repository's maintainer |
| 7 | `mode = "enforce"` | Every repository completes its seven-day scan with zero findings in authored commits and in titles recovered from merge-commit subjects. The operator lists open pull requests whose titles fail with the proposed base mode and corrects them before the flip. Each open title passes on a rerun. | Operator, rules maintainer |

## Appendices

- [Decisions](decisions.md) records the operator's answers and the design decisions.
- [A. Evidence](A-evidence.md): the subject counts and the current code, each labeled.
- [B. Alternatives considered](B-alternatives.md): the rejected policies, homes, and mechanisms, with reasons.
- [C. Configuration](C-configuration.md): the file formats and the loader contract. It also lists the global seed and how reviewers check it.
- [D. Checker](D-checker.md): arguments, messages, exit codes, failure modes, and the test plan.
