---
type: Design Appendix
title: Commit types and scopes decisions
description: The open and decided questions of the commit types and scopes design.
tags: [commits, decisions, design]
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

# Commit types and scopes decisions

Back to [the design](README.md).

## Open

None.

## Decided

### D10. Which commit check governs the merge?

- **Decided.** Coordination chose option A on 2026-11-23 in the design review follow-up. The choice is reversible.
- **Rule.** In `enforce` mode, the checker blocks titles with scope allow-list or per-type scope findings and reports advisories for authored commits. Existing syntax findings keep their behavior.
- **Merge gate.** CI checks the title on `pull_request` events `opened`, `edited`, `synchronize`, and `reopened`. The ruleset requires the `conclusion` check, which requires a successful title check and fails on skipped or missing results. `tbhbdev alpha merge` rechecks the latest title with D7's fetched base mode for early warning before writing the merge subject.
- **Why.** The GitHub merge button and GitHub auto-merge bypass `alpha merge` and need the required CI check to block a merge on title findings. A merge can still include a commit with an unlisted scope. [Renovate auto-merge policy](../../adrs/0024-renovate-pull-requests-automerge-after-approval-and-ci.md)
- **Accepted technical debt (`TECH-DEBT`).** During the `evaluate` to `enforce` flip, GitHub can accept a passing title check on a head behind the base. The merge command can also fetch the base before the flip and submit the merge after it. The operator checks [open pull requests before the flip](README.md#rollout).
- **Accepted technical debt (`TECH-DEBT`).** After a title-only edit, the ruleset accepts the previous passing `conclusion` for the same head commit until the run for `edited` registers its replacement a few seconds later. The GitHub merge button or auto-merge can land a title that fails the checker during that window, even in steady-state `enforce` mode. `alpha merge` narrows the gap for its path by rechecking the live title. Option B prevents those merges by requiring the merge queue to check the title on `merge_group`.
- **Upgrade.** Move to option B when the merge queue runs the commit check on `merge_group` as a required check. The queue checks against the base it uses for the merge, resolving the race and blocking authored-commit scope findings in `enforce` mode. [B. Alternatives](B-alternatives.md#where-to-enforce-scopes)
- **Undo.** Sync `mode = "evaluate"` to the base, then update open heads that still declare `enforce` to make their title scope findings advisory. To roll back option B, restore advisories for authored-commit scope findings in `enforce` mode.
- **Record.** This design's pull request.

### D3. Who may add a scope to the global list?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** The rules maintainer approves new global scopes, including the 16-scope seed, through ordinary rules pull requests.
- **Why:** Reviewers can check each addition against the [scope class tests](README.md#scope-classes).
- **Undo:** The operator can revoke the rules maintainer's authority to approve additions.
- **Record:** This design's pull request.

### D2. Adopt the scope policy for each type?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** `build` and `ci` accept only global scopes, and every other type accepts global scopes and that repository's scopes. No type requires a scope, and a missing scope always passes.
- **Why:** The policy keeps global scopes such as `deps` and `mcp` available to `feat` and `fix` without forcing writers to invent scopes.
- **Undo:** Change the policy in a rules pull request and sync the global file.
- **Record:** This design's pull request.

### D9. Is the breaking marker allowed on every type?

- **Decided:** 2026-11-23, by the rules maintainer, in this design.
- **Rule:** `!` before the colon passes on any type and scope that the allow list accepts. Titles follow the same rule. It counts toward the length limit. The history shows it only on `feat`, 6 times.
- **Undo:** add a `breaking` flag for each type to the schema and the global file. Two-way door.
- **Record:** this design's pull request.

### D8. Where does the rollout mode live?

- **Decided:** 2026-11-23, by the rules maintainer, in this design.
- **Rule.** `mode = "evaluate" | "enforce"` goes in the global file, and `#Repo` rejects it in a repository file. Scope allow-list and per-type scope findings for titles block in `enforce` mode, and those for authored commits remain advisory in both modes under D10. Existing blocking commit checks and the title cap apply in both modes. [D. Checker](D-checker.md#exit-codes)
- **Undo.** Set `mode = "evaluate"` again in one rules pull request and sync it. Update any open heads still in `enforce` as described in D7.
- **Record:** this design's pull request.

### D7. Which commit's configuration does a pull request check read?

- **Decided:** 2026-11-23, by the rules maintainer, in this design.
- **Rule.** Continuous integration (CI) and the merge command read the scope lists from the exact head of the pull request so a change can add and use a scope together. Both apply the stricter of the head and base modes to the title, with CI using the event's base and the merge command fetching the live base. CI checks authored commits using the head's scope lists and reports scope findings as advisories under D10. [D. Checker](D-checker.md#pull-request-configuration)
- **Rollback.** An open head that still declares `enforce` keeps blocking title scope findings after the base returns to `evaluate`. Both revisions must declare `evaluate` for those findings to become advisory.
- **Undo.** Pass the base branch's files to the loader. Two-way door.
- **Record.** This design's pull request.

### D6. How do scope spellings outside the grammar resolve?

- **Decided:** 2026-11-23, by the rules maintainer, in this design.
- **Rule.** A listed scope matches `^[a-z][a-z0-9]*(-[a-z0-9]+)*$` and has at most 24 characters, and the schema rejects any other key. The checker flags `_`, backtick-wrapped scopes, `snake_case` scopes, file paths, and a comma list of scopes, and suggests the nearest listed spelling. An identifier such as `e-000001` fits the grammar but still produces an unlisted-scope finding, because a scope refers to a part of the repository, never to one record.
- **Undo:** widen the pattern in the schema. Two-way door.
- **Record:** this design's pull request. [A. Evidence](A-evidence.md#scope-spellings) counts the spellings.

### D5. Does `sync` become a type?

- **Decided:** 2026-11-23, by the rules maintainer, in this design.
- **Rule:** no. The 11 types are unchanged. The one repository whose automation writes `sync` moves it to `chore(sync)` and lists `sync` as a repository scope.
- **Undo:** add `sync` to the global file's types. Two-way door.
- **Record:** this design's pull request.

### D4. Where do the files live?

- **Decided:** 2026-11-23, by the rules maintainer, in this design.
- **Rule:** the sources are `repo/baseline/files/tbhb/global/<section>.toml` and `repo/baseline/files/tbhb/schema/<section>.cue` in rules. The baseline copies them to `.tbhb/global/<section>.toml` and `.tbhb/schema/<section>.cue` in mode `bytes`, and each repository writes its own `.tbhb/<section>.toml`. The schema files declare `package schema`, and nothing imports them, so they don't need a Configure Unify Execute (CUE) module path. The brief's `config/global/` and `config/schema/` moved because the manifest schema rejects a source outside `repo/baseline/`.
- **Undo:** move the files and their manifest entries. Two-way door until other repositories read the paths, then a coordinated move.
- **Record:** this design's pull request, and the manifest schema in [A. Evidence](A-evidence.md#the-rules-repository-at-a05c418).

### D1. Do commit types and scopes come from an allow list?

- **Decided:** 2026-11-23, by the operator.
- **Rule.** Commit subjects and pull request titles must use a listed type. The checker evaluates any scope against the allow list and per-type policy. Scope findings block titles in `enforce` mode and remain advisory for authored commits under D10.
- **Undo:** the operator decides again, and the checker stops reading the type and scope lists.
- **Record:** the operator's decision of 2026-11-23, relayed in the brief that commissioned this design.
