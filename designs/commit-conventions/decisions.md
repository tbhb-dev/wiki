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
- **Rule.** `mode = "evaluate" | "enforce"` goes in the global file, and `#Repo` rejects it in a repository file. Evaluate mode makes only the scope allow-list and per-type scope findings advisory. Existing commit checks and the title cap block in both modes. [D. Checker](D-checker.md#exit-codes) defines the exit codes.
- **Undo:** set `mode = "evaluate"` again in one rules pull request and sync it. Two-way door.
- **Record:** this design's pull request.

### D7. Which commit's configuration does a pull request check read?

- **Decided:** 2026-11-23, by the rules maintainer, in this design.
- **Rule.** CI and the merge command read the scope lists from the exact pull request head SHA. This lets a pull request add a scope and use it in the same change. Both apply the stricter of the head mode and the base branch mode. [D. Checker](D-checker.md#pull-request-configuration) specifies how the callers resolve both revisions.
- **Undo.** Pass the base branch's files to the loader. Two-way door.
- **Record.** This design's pull request.

### D6. How do scope spellings outside the grammar resolve?

- **Decided:** 2026-11-23, by the rules maintainer, in this design.
- **Rule:** a listed scope matches `^[a-z][a-z0-9]*(-[a-z0-9]+)*$` and has at most 24 characters, and the schema rejects any other key. The checker rejects `_`, backtick-wrapped scopes, `snake_case` scopes, file paths, and a comma list of scopes, and suggests the nearest listed spelling. An identifier such as `e-000001` fits the grammar but fails as unlisted, because a scope refers to a part of the repository, never to one record. History keeps every old spelling.
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
- **Rule:** yes. The type and scope of a commit subject must appear on an allow list. This design defines the list and how a check enforces it.
- **Undo:** the operator decides again, and the checker stops reading the type and scope lists.
- **Record:** the operator's decision of 2026-11-23, relayed in the brief that commissioned this design.
