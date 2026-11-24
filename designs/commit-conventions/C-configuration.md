---
type: Design Appendix
title: Commit types and scopes configuration
description: The global and repository files, their schema, the shared loader contract, and the global seed list.
tags: [commits, configuration, design]
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
<!-- cspell:ignore pnpm rulesets snake evals afenc autosquash -->

# Commit types and scopes configuration

Back to [the design](README.md).

Everything here is new and proposed, and none of it exists today.

## The loader contract

Every section that the loader serves follows one contract. The commits section and the section for the issue taxonomy share the loader and the file layout.

- **Signature.** `load(section: str, repo_root: Path) -> Loaded`. `section` matches `^[a-z][a-z0-9-]*$`, and the loader rejects any other value before it touches a path.
- **Files.** The global file is `<repo_root>/.tbhb/global/<section>.toml`, the schema is `<repo_root>/.tbhb/schema/<section>.cue`, and the repository file is `<repo_root>/.tbhb/<section>.toml`.
- **Schema.** Each schema file declares `package schema` and two closed definitions, `#Global` and `#Repo`. `#Repo` holds `schema_version` and only the tables a repository may extend.
- **Extendable tables.** Each table that `#Repo` declares maps a name to a one-line description. The commits schema rejects control characters and line and paragraph separators in names and descriptions. The loader merges by key, global entries first, and records each key's class as `global` or `repo`.
- **Result.** `Loaded.values` holds the merged settings. `Loaded.classes` records the class of each key in the extendable tables, and `Loaded.notices` lists the duplicate keys.
- **Errors.** The loader raises `ConfigError(path, finding)` for a missing or unreadable global file, a missing schema, a missing `cue` binary, any schema finding, a TOML parse error, and a repository file that exists but that the loader can't read. An absent repository file isn't an error.
- **Versions.** Both files must declare `schema_version = 1`. The schema rejects a missing or different version. A version change releases a new schema and global file together.

The loader doesn't contain section-specific code. A new section needs only its three files.

## The global file

The rules source is `repo/baseline/files/tbhb/global/commits.toml`. The seed below lists every value the checker needs. The checker's code doesn't define any list.

```toml
schema_version = 1

# Scope findings block titles in enforce mode.
# Authored-commit scope findings are advisory in both modes.
mode = "evaluate"

[limits]
subject = 72 # characters in a commit subject
title = 64   # leaves room for " (#9999)" in a 72-character merge subject

# Policy per type: none, global, repo, or any.
[types]
build = { scopes = "global", description = "Build system, toolchain, and pins." }
chore = { scopes = "any", description = "Upkeep that changes no behavior." }
ci = { scopes = "global", description = "CI workflows and their settings." }
docs = { scopes = "any", description = "Documentation only." }
feat = { scopes = "any", description = "New behavior a user can see." }
fix = { scopes = "any", description = "A corrected behavior." }
perf = { scopes = "any", description = "Faster or smaller, same behavior." }
refactor = { scopes = "any", description = "Restructured code, same behavior." }
revert = { scopes = "any", description = "Undoes an earlier commit." }
style = { scopes = "any", description = "Formatting only." }
test = { scopes = "any", description = "Tests only." }

[scopes]
agents = "Agent instructions and harness configuration."
bazel = "Bazel modules, settings, and the Bazel version."
deps = "Dependencies and lockfiles, mostly from Renovate."
deps-dev = "Development-only dependencies, from Renovate."
gitleaks = "Secret-scanning configuration."
go = "The Go toolchain and Go module settings."
hooks = "Git hooks and their configuration."
lint = "Lint gates and their configuration."
mcp = "The repository's Model Context Protocol server or client configuration."
mise = "Tool versions that mise pins."
pnpm = "The pnpm toolchain and workspace settings."
python = "The Python toolchain and project settings."
renovate = "Renovate configuration."
rulesets = "GitHub rulesets the baseline supplies."
uv = "The uv toolchain and its lockfile."
workflows = "GitHub Actions workflow files."

[subjects]
# Git writes these commit subjects with a quoted subject. They pass unchanged.
passthrough = ["Revert", "Reapply"]
# Git's autosquash markers, which must be reworded before review.
rewrite_prefixes = ["fixup!", "squash!", "amend!"]
# Markers of unfinished work, matched as whole words in any case.
unfinished = ["wip"]
# Both modes normalize and reject generic summaries.
generic = [
  "address feedback", "address review", "address review comments", "change",
  "changes", "fix", "fixes", "misc", "review feedback", "tweak", "tweaks",
  "update", "updates", "wip",
]
# These prefixes lift the length cap for commit subjects only.
long_prefixes = [
  "chore(deps): update ", "chore(deps): pin ",
  "chore(deps-dev): update ", "chore(deps-dev): pin ",
  "fix(deps): update ", "fix(deps): pin ",
  "fix(deps-dev): update ", "fix(deps-dev): pin ",
]
```

`limits.title` caps the title before GitHub appends a space and `(#N)`. The seed leaves 8 characters between the 64-character title cap and the 72-character merge subject cap, enough for four-digit pull request numbers, while five digits need 9 characters. The schema requires `limits.title <= limits.subject`, and the merge command checks the final length with the actual suffix.

## A repository file

Each repository writes `.tbhb/commits.toml` with only the scopes that name its own parts.

```toml
schema_version = 1

[scopes]
alpha = "The alpha command group."
designs = "Design documents under designs/."
sync = "The automation that copies shared files in."
```

## The schema

The rules source is `repo/baseline/files/tbhb/schema/commits.cue`.

```cue
package schema

import "strings"

#OneLine:     !~"[\\p{Cc}\\p{Cf}\\p{Zl}\\p{Zp}]"
#Name:        #OneLine & =~"^[a-z][a-z0-9]*(-[a-z0-9]+)*$" & strings.MaxRunes(24)
#Description: #OneLine & =~"\\S" & strings.MaxRunes(120)

#Global: {
	schema_version!: 1
	mode:            "evaluate" | "enforce"
	limits: {
		subject: int & >=40 & <=100
		title:   int & >=40 & <=subject
	}
	types: [#Name]: {
		scopes:      "none" | "global" | "repo" | "any"
		description: #Description
	}
	scopes: [#Name]: #Description
	subjects: {
		passthrough: [...string & =~"\\S"]
		rewrite_prefixes: [...string & =~"\\S"]
		unfinished: [...string & =~"\\S"]
		generic: [...string & =~"\\S"]
		long_prefixes: [...string & =~"\\S"]
	}
}

#Repo: {
	schema_version!: 1
	scopes?: [#Name]: #Description
}
```

Both definitions are closed, so a repository file that sets `mode`, `limits`, `types`, or `subjects` fails `cue vet`. A repository can't remove a global scope, because it has no way to name one for removal.

### Schema verification

Save the samples above as `commits.cue`, `global.toml`, and `repo.toml`. Run these commands with pinned CUE 0.17.1.

```bash
cue vet -c -d '#Global' commits.cue global.toml
cue vet -c -d '#Repo' commits.cue repo.toml
sed '/^schema_version/d' repo.toml > missing-version.toml
cue vet -c -d '#Repo' commits.cue missing-version.toml
sed 's/^title = 64/title = 73/' global.toml > title-too-long.toml
cue vet -c -d '#Global' commits.cue title-too-long.toml

for code in 0000 0009 000A 007F 0085 200D 2028 2029; do
  printf '%s\n' 'schema_version = 1' '[scopes]' \
    "alpha = \"First\\u${code}second\"" > "description-$code.toml"
  printf '%s\n' 'schema_version = 1' '[scopes]' \
    "\"alpha\\u${code}beta\" = \"One line.\"" > "scope-$code.toml"
  cue vet -c -d '#Repo' commits.cue "description-$code.toml"
  cue vet -c -d '#Repo' commits.cue "scope-$code.toml"
done
```

The two original samples pass. The missing version and the title limit above the subject limit fail. The bad descriptions and scope names fail for ISO control characters, Unicode format controls, and Unicode line and paragraph separators.

## The global seed

The seed holds 16 scopes. Each one passes the class tests in [the README](README.md#scope-classes). A shared source provides it, and it means the same thing everywhere without being a repository name.

| Scope | What provides it | Repositories using it in 60 days |
| --- | --- | --- |
| `deps` | The semantic commits that Renovate writes | 36 |
| `mcp` | The baseline's `.mcp.json` and its Model Context Protocol placeholders | 21 |
| `renovate` | The shared Renovate configuration and workflows | 18 |
| `mise` | The baseline's `mise.toml` | 16 |
| `agents` | The baseline's agent settings, with each repository's agent instructions | 8 |
| `workflows` | The baseline's workflow files | 6 |
| `python`, `go`, `pnpm` | The baseline kinds and the rules toolchains | 4 each |
| `lint` | The rules lint gates | 3 |
| `hooks` | The baseline's hook configuration | 3 |
| `rulesets` | The baseline's `.github/rulesets/main.json` | 3 in the spot check |
| `gitleaks` | The baseline's `.gitleaks.toml` | 3 in the spot check |
| `bazel` | The baseline's Bazel files | seen backtick-wrapped |
| `uv` | The lockfile tool of the Python kind | 2 subjects in the spot check |
| `deps-dev` | The development dependency subjects that Renovate writes, which the current checker already exempts | not counted |

### Scopes left out of the seed

| Scope | Repositories | Why it stays out |
| --- | --- | --- |
| `ci`, `docs` | 10, 4 | Each repeats a type. Write `ci: …` or `docs: …` instead. |
| `tools`, `ops`, `research`, `evals`, `tokens`, `ansible`, `awrel`, `afenc` | 3 to 10 | In some repositories each value refers to another project, and in others to a local part, which fails class test 2. A repository that writes about another project lists that value itself. |
| `deploy` | 5 | Its meaning differs by repository. |
| `agent` | 4 | It refers to a local component, which differs from `agents`. |
| `claude` | 3 | It refers to one harness's files, which `agents` covers for every harness. A repository that builds a feature for that harness lists it. |
| `release` | 3 | The baseline doesn't provide a shared release mechanism. |
| `_`, backtick-wrapped, `snake_case`, paths, record identifiers | not applicable | They fail the grammar or refer to one record. [Decision D6](decisions.md#d6-how-do-scope-spellings-outside-the-grammar-resolve) covers them. |

## Reviewing the seed lists

- **The global seed** arrives in plan step 1 as a rules pull request. Its body lists each scope with its class-test reasons and its counts from appendix A, and it takes the design review pair. [Decision D3](decisions.md#d3-who-may-add-a-scope-to-the-global-list) settles who approves it.
- **Each repository's seed** comes from a report over its last 60 days in `evaluate` mode, in plan step 6. The maintainer lists only the scopes that refer to the repository's own parts or another repository it writes about, and the pull request body gives the report's counts for each one. The repository's normal review applies.
- **Later additions** follow the same two paths. A scope that turns up in a repository file in three or more repositories is a candidate for a global review, and moves only if it passes the class tests.
