---
type: Design Appendix
title: Commit types and scopes from an allow list configuration
description: The global and repository files, the schema, the merge rule, and the loader's failure cases for the commits section.
tags: [commit-conventions, config, schema, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore evals -->

# Commit types and scopes from an allow list configuration

Back to [the design](README.md).

Everything here is new and planned. The files use TOML, Tom's Obvious Minimal Language, and the schema uses Configure Unify Execute (CUE).

## Paths

| File | Repository | Owner |
| --- | --- | --- |
| `config/global/<section>.toml` | tbhb-dev/rules | Rules maintainer |
| `config/schema/<section>.cue` | tbhb-dev/rules | Rules maintainer |
| `.tbhb/<section>.toml` | Each repository, optional | That repository's maintainer |

For commits, `<section>` is `commits`. The CUE module file `cue.mod/module.cue` in tbhb-dev/rules declares `module: "tbhb.dev/rules"`, and each schema file declares `package schema`, so its import path is `tbhb.dev/rules/config/schema`. The `.tbhb/` directory holds only these section files, so a repository's tbhb-dev configuration sits in one place apart from tool configuration at the root.

## The global file

```toml
version = 1
mode = "evaluate"
max_subject = 72
max_title = 64

passthrough_prefixes = ['Revert "', 'Reapply "']
rewrite_prefixes = ["fixup!", "squash!", "amend!"]
forbidden_words = ["wip"]
generic_summaries = [
  "address feedback", "address review", "address review comments",
  "change", "changes", "fix", "fixes", "misc", "review feedback",
  "tweak", "tweaks", "update", "updates", "wip",
]
length_exempt = [
  "chore(deps): update ", "chore(deps): pin ",
  "chore(deps-dev): update ", "chore(deps-dev): pin ",
  "fix(deps): update ", "fix(deps): pin ",
  "fix(deps-dev): update ", "fix(deps-dev): pin ",
]

[scopes]
global = [
  "deps", "deps-dev", "mcp", "renovate", "mise", "tools", "ci", "agents",
  "docs", "python", "go", "pnpm", "ansible", "release", "lint", "hooks", "bazel",
]

[types.build]
classes = ["global"]

[types.ci]
classes = ["global"]

[types.feat]
classes = ["global", "repo"]

# chore, docs, fix, perf, refactor, revert, style, and test
# repeat the feat entry.
```

Every value the check compares against sits in this file. The code keeps only the grammar of a subject line and the order of the checks.

## A repository file

```toml
version = 1

[scopes]
repo = ["evals", "lab"]

[types.ci]
scopes = ["evals"]
```

This file adds two repository scopes, which every type with the `repo` class then accepts, and lets `ci` take `evals` as well as the global scopes.

## The schema

```cue
package schema

#Scope: =~"^[a-z][a-z0-9]*(-[a-z0-9]+)*$"
#Class: "global" | "repo"

#TypeRule: {
	classes?: [...#Class]
	scopes?: [...#Scope]
}

#Rules: {
	version:     1
	mode:        "evaluate" | "enforce"
	max_subject: int & >0
	max_title:   int & >0 & <=max_subject
	passthrough_prefixes: [...string]
	rewrite_prefixes: [...string]
	forbidden_words: [...string]
	generic_summaries: [...string]
	length_exempt: [...string]
	types: [=~"^[a-z]+$"]: #TypeRule & {classes: [...#Class]}
}

#Global: {
	#Rules
	scopes: {global: [...#Scope]}
}

#Repo: {
	version: 1
	scopes?: {repo: [...#Scope]}
	types?: [string]: #TypeRule
	generic_summaries?: [...string]
}

#Merged: {
	#Rules
	scopes: {
		global: [...#Scope]
		repo?: [...#Scope]
	}
}
```

CUE closes each definition, so an unknown key is an error, and a repository file that sets `mode`, `max_title`, or `scopes.global` fails `#Repo`. This schema is a sketch, and plan step 1 checks it with `cue vet` against both example files. The loader also checks that every type key in the repository file exists in the global file, which `#Repo` alone can't see.

## The merge rule

The loader handles every section the same way, so the issue taxonomy's section follows the same steps:

1. Read the global file from the `runfiles` of the calling binary. If it's missing or unreadable, stop with exit 2.
1. Read the repository file if it exists. If the loader can't read or parse an existing file, stop with exit 2.
1. Run `cue vet` on the global file against `#Global` and on the repository file against `#Repo`. Any error stops with exit 2, and the message names the file and the field.
1. Merge. Each list present in both files becomes the global values followed by the repository values not already present. The loader copies a key present in one file only as it stands.
1. Run `cue vet` on the merge against `#Merged` and check the type keys. Any error stops with exit 2.

A repository's values only ever extend a list. No key lets a repository remove a value, and `#Repo` has no scalar a repository could change.

## The loader's interfaces

| Interface | Where | Returns |
| --- | --- | --- |
| The Python library `tbhb_config.load(section, repo_root)` | tbhb-dev/rules, in the `runfiles` of `check_subject` | The merged section as a dict, or raises `ConfigError` |
| The binary `tbhb_config <section> --repo-root DIR` | tbhb-dev/rules, exposed as `.bazel_env/bin/tbhb_config` | The merged section as JSON on stdout and exit 0, or the error on stderr and exit 2 |
| The function `load(section, repo_root)` | tools | Runs the binary and parses its output, and raises on exit 2 |

The global file reaches a repository through the binary's `runfiles`, at that repository's pinned rules version. A change to the global list lands everywhere through the usual rules version bump, and each bump is a reviewed pull request.

## Failure cases

| Case | Result |
| --- | --- |
| The global file is missing | Exit 2, both modes |
| A file fails its schema | Exit 2. The message names the file and the field. |
| The repository file is missing | Normal. The repository adds nothing. |
| The loader can't read or parse an existing repository file | Exit 2 |
| The repository file sets a key only the global file may set | Exit 2 |
| The repository file names a type the global file lacks | Exit 2 |
| A listed scope fails the grammar | Exit 2 |
