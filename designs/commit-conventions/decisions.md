---
type: Design Appendix
title: Commit types and scopes from an allow list decisions
description: The open and decided questions of the commit conventions design.
tags: [commit-conventions, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore evals -->

# Commit types and scopes from an allow list decisions

Back to [the design](README.md).

## Open

These three items set standing rules of the operator's own for every commit in the organization, so they wait for the operator.

### D2. Adopt the seed global scope list in appendix C as written?

- **Recommendation:** yes, the 17 scopes in [C. Seed lists](C-seed-lists.md#proposed-global-scopes), with `workflows`, `deploy`, `ops`, `agent`, and `claude` left to repository files.
- **Why:** Each listed scope names a tool, a dependency channel, or a baseline concern that means the same thing everywhere, and together they cover the 863 `deps` subjects and every scope used in 10 or more repositories.
- **If yes:** Plan step 1 ships the list in `evaluate` mode, and the rules maintainer adds or promotes scopes from the `evaluate` reports.
- **If no:** The operator names the scopes to add or drop, and step 1 ships that list instead.
- **Reversible:** Yes. A rules release edits the list, and repositories pick it up at their next rules bump.

### D3. Adopt the per-type policy: `build` and `ci` take global scopes only, every other type takes global and repository scopes, and no type requires a scope?

- **Recommendation:** yes, as in the README's [per-type table](README.md#per-type-allow-lists).
- **Why:** Build and CI changes edit toolchain pins and workflows, which the global scopes name, and 81% of `build` subjects carry no scope at all. Restricting `feat` and `fix` to repository scopes would refuse the Renovate `fix(deps)` and shared work such as `feat(mcp)`, and every type has a large share of subjects with no scope today, from 22% for `feat` to 81% for `build`.
- **If yes:** The global file lists the classes in the table, and a missing scope passes for every type.
- **If no:** The operator names a type to make scope-free or to require a scope, and the global file sets it. A scope-free type has an empty `classes` list, and a required scope needs a `scope_required` flag in the schema.
- **Reversible:** Yes. It's one table in the global file.

### D4. Keep the type list global, so no repository can add a type, and write `sync` as `chore(sync)`?

- **Recommendation:** yes.
- **Why:** The 11 types are the operator's standing list and the ones `cog verify` accepts, and `sync` appears in one repository only. A type list each repository could extend would let each repository grow its own vocabulary, which is the problem the allow list fixes for scopes.
- **If yes:** `#Repo` rejects a `types` entry for a type the global file lacks, and the one repository adds `sync` to its own scopes.
- **If no:** `#Repo` accepts new types, and each such repository also configures `cog` to accept them, because `cog verify` rejects an unknown type.
- **Reversible:** Yes, by a schema change in a rules release.

## Decided

### D10. The loader is one implementation in tbhb-dev/rules

- **Decided:** 2026-11-23, by the rules maintainer role in this design.
- **Rule:** The merge and validation code lives in tbhb-dev/rules beside the global files and the schema, ships in the `runfiles` of `check_subject` and of a `tbhb_config` binary, and tools' `load(section, repo_root)` runs that binary and parses its JSON output.
- **Undo:** Move the code into the tools package and give `check_subject` a copy of the global file, at the cost of two loaders that can drift.
- **Record:** This design, [B. Alternatives considered](B-alternatives.md#two-loaders).

### D9. Rollout runs in `evaluate` mode first, and nothing rewrites history

- **Decided:** 2026-11-23, by the rules maintainer role in this design.
- **Rule:** The global `mode` starts at `evaluate`, where the check reports scope findings without failing, and moves to `enforce` after 14 days with no unfixed report. The check runs on new commits and titles only, and existing subjects on `main` stay as written.
- **Undo:** Set `mode = "enforce"` earlier, or back to `evaluate`, in a rules release.
- **Record:** This design, [README rollout](README.md#rollout-and-history).

### D8. `check_subject` exits 0, 1, or 2, and fails closed on configuration

- **Decided:** 2026-11-23, by the rules maintainer role in this design.
- **Rule:** Exit 0 means pass, 1 means subject problems, and 2 means a usage error or a configuration that failed closed. Exit 2 applies in both modes.
- **Undo:** Change the codes in the same release as every caller.
- **Record:** This design, [README subject check](README.md#the-subject-check).

### D7. Pull request titles take the same check with a 64-character limit

- **Decided:** 2026-11-23, by the rules maintainer role in this design.
- **Rule:** `--title` applies every subject rule with `max_title = 64`, the existing tools limit, and every type accepts the `!` marker, which counts toward the length.
- **Undo:** Drop the CI title step and the `--title` calls.
- **Record:** This design, and `MAX_TITLE` in the tools pull request command, cited in [A. Evidence](A-evidence.md).

### D6. Scopes are lowercase hyphenated words, and the oddities don't enter any list

- **Decided:** 2026-11-23, by the rules maintainer role in this design.
- **Rule:** The schema requires `^[a-z][a-z0-9]*(-[a-z0-9]+)*$` for every listed scope, which excludes `_`, backticks, and paths. Identifiers such as `e-000001` match the grammar but go in the summary, not on a list.
- **Undo:** Widen the pattern in the schema.
- **Record:** This design, [C. Oddities](C-seed-lists.md#oddities).

### D5. The proposed paths stand, with the schema package named `schema`

- **Decided:** 2026-11-23, by the rules maintainer role in this design.
- **Rule:** The files are `config/global/commits.toml` and `config/schema/commits.cue` in tbhb-dev/rules, and `.tbhb/commits.toml` in a repository. The Configure Unify Execute (CUE) module is `tbhb.dev/rules`, the files declare `package schema`, and `tbhb.dev/rules/config/schema` is the import path, because a package clause takes an identifier, not a path.
- **Undo:** Rename the files in a rules release and the loader's path constants with them.
- **Record:** This design, [D. Configuration](D-configuration.md).

### D1. Commit types and scopes come from an allow list

- **Decided:** 2026-11-23, by the operator.
- **Rule:** Types and scopes come from an allow list, global values merge with each repository's own, a repository extends the list and never narrows it, the configuration holds every value list, and a missing global file, a schema error, or an unreadable repository file fails closed.
- **Undo:** A new ruling by the operator.
- **Record:** The operator's ruling of 2026-11-23, restated in this design's pull request.
