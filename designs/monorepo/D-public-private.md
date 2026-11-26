---
type: Design Appendix
title: One public monorepo for the factory public and private split
description: What the private companion holds, who carries out each duty in each repository, how public code reads private values at build and deploy time, and the scan rules that keep private content out of the public repository.
tags: [monorepo, privacy, gitleaks, bazel, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo tfvars tbhbdev evals -->

# One public monorepo for the factory public and private split

Back to [the design](README.md).

## What goes where

The companion is `tbhbdev-private`, and it's the only private repository in the pair (D20). It mirrors the public paths, so a private file sits at the same path as the public code that reads it.

| Source today | Public monorepo | Private companion |
| --- | --- | --- |
| ops | Ansible playbooks and the task sets they include, service definitions, dashboards, the ops packages, OpenTofu modules, and the schema for inventory and values | The inventory, group and host variables, OpenTofu variable files and backend settings, and the overlay network's configuration |
| agent-config | Agent and ops configuration, routing, schemas, and docs | The operator-account folder |
| agent-memories | Memories marked public | Every other memory |
| wiki, research, spikes, evals, tools | The public repositories as they are | Their private counterparts, at `docs/wiki`, `docs/research`, `docs/spikes`, `factory/evals`, and the tool's folder |
| sm | Code and docs that pass the scan | Anything the scan holds back |
| New | A stub of the companion with fake values | The private-name list, and each import's scan report |

The companion holds values and prose. It holds no build rules, no service code, and no copy of public files. If a private file needs logic, the logic goes in the public repository and reads the value.

Secrets stay out of both repositories. They remain in 1Password and reach a process at run time, as they do today (existing).

A memory is public only when its front matter says `visibility: public` and it passes the scan. The default is private, so a forgotten field keeps a memory out.

A project's needs, requirements, and knowledge articles follow the same split. The public ones sit in the project's own `docs/` folder in the monorepo, such as `factory/tbhbdev/docs/kb`, and one that needs a private detail sits at the same path in the companion (D33).

## Who carries out each duty in each repository

The duties and the identities are the same on both sides ([appendix C](C-layout-and-ownership.md#separation-of-duties)). The split changes only what each repository's merge path looks like.

| Duty | Public monorepo | Private companion |
| --- | --- | --- |
| Build | `tbhb-agent`, `tbhb`, or the Renovate App | The same three. The Renovate App's run there moves the companion's one pin |
| Review | `tbhb-agent`, in a fresh run, with the verdict published through the reviews App | The same, on a copy clone |
| Approve | The reviews App at tiers 1 and 2, and the tier 0 workflow at tier 0 | The same, with the tiers read from the monorepo commit the companion pins |
| Merge | `tbhb-agent` or `tbhb`, through GitHub's merge queue | `tbhb-agent` or `tbhb`, on a branch that contains `main`'s tip |
| Publish private-origin code | Two independent tier 2 `tbhb-agent` runs clear the import before push (D4) | Holds the report and both verdicts |

Each project's `OWNERS` file exists on both sides at the same path. The companion's copy describes only what the companion holds for the project, and it names the same two identities.

## How public code reads private values

### At build time

The companion is a Bazel module named `tbhb_private`. The public repository holds a stub of that module at `third_party/private_stub`, with the same target names and fake values that match the schema.

```starlark
# MODULE.bazel in the public monorepo
bazel_dep(name = "tbhb_private")
local_path_override(module_name = "tbhb_private", path = "third_party/private_stub")
```

```sh
# on a host that deploys, with the companion checked out beside the monorepo
bazel run //factory/ops:deploy --override_module=tbhb_private=../tbhbdev-private
```

Public targets refer to `@tbhb_private//factory/ops:inventory` and get the stub unless a command overrides it. Whether the command-line override wins over the one in `MODULE.bazel` is unverified, and plan step 2 tests it.

The public repository owns the schema for every private value. The stub must validate against it in public CI, and the companion's values must validate against it in the companion's CI. A schema change that breaks the real values fails in the companion before a deploy can use it.

### At deploy time

The rule is that the public repository builds and tests, and the companion deploys.

1. The companion pins one monorepo commit in a single file. This is the only first-party pin left in the factory, and it points at the new repository, never at an old one.
2. The companion's workflow checks out both repositories, passes the override, and runs the deploy target on private runners.
3. Deploy logs stay in the private repository. Public CI logs never print a real value, because public CI never has one.

A deploy that needs only secrets, such as a Cloudflare Worker, can run from the public repository under a protected environment limited to `main`. A deploy that needs any inventory or host value runs from the companion.

### On an agent host

`tbhbdev` finds the companion through one setting, which defaults to a checkout beside the monorepo. A host without access to the companion runs every public build and test. A command that needs a private value fails with a message that names the missing setting.

## Keeping private content out

A branch pushed to a public repository is public at once, even before any review. Scanning in CI is therefore too late on its own, and the first barrier has to run on the host.

| Layer | Where it runs | What it catches | State |
| --- | --- | --- | --- |
| gitleaks with the shared rules, including the public address rule | Host commit hook, and CI on every commit of a pull request | Secrets and public addresses | existing |
| Private-name scan | Host commit and push hooks, reading the list from the companion checkout | Host names, overlay network names, internal addresses, private repository names, and internal paths | new |
| The same scan in CI | A job on pull requests with access to the repository secret and on each queue candidate (D6) | A commit from a host whose hook didn't run | new |
| Import scan and review | The import tool and two fresh agent review runs, before the first public push (D4) | Private content in a private-origin repository | new |
| Review | The reviewer of each pull request | What a pattern can't catch | existing |

The companion stores the private-name list. Under the organization administrator grant, the repository stores a copy in a GitHub Actions environment secret and restricts that environment's deployment rule to the `main` branch and the base branch workflow. A `pull_request` or `merge_group` run of an agent-edited `ci.yml` has no access to the secret. A queue candidate's ref isn't `main`, so scanning it with the secret depends on the base branch workflow reaching the candidate, which plan step 2 tests. Until then a candidate relies on the scan of its pull request's head and the host hooks.

The trusted scan reads candidate files as data and never runs their code. It prints a file and line for each finding and omits the match. It scans fork changes at the queue candidate.

The name scan on a host needs the companion. The hook installer blocks pushes to the monorepo from hosts that have a private checkout but lack the companion.

Hooks come from `tbhbdev alpha install` only, as today (existing rule). The push hook is new, and it scans every commit that the push would publish.

## Importing a private-origin repository

1. In the private repository, a pull request adds a path map: which globs are public and which go to the companion. It gets its review there, before anything is copied.
2. The import tool builds the public tree from the map, and the companion tree from the rest.
3. The tool makes one signed public copy commit in a private workspace. It brings no source history. The companion tree goes in its own commit in the companion.
4. The tool runs gitleaks and the private-name scan on the candidate and records the results in the companion. The report lists every public file, every finding, and every held-back file. It names the exact copy commit and the manifest digest.
5. Before the first public push, two fresh tier 2 `tbhb-agent` runs with private access independently read every public file, verify the held-back list against the frozen source and path map, and check every finding and its disposition. Both runs use a provider other than the builder's and share no context with the builder or each other. They record verdicts for the exact commit and report in the companion.
6. The import tool blocks the public push until both verdicts clear the same candidate and report and the scans and private comparison pass. An incomplete report, unresolved finding, or missing, failed, or stale check or verdict blocks publication. A changed commit or report goes through the gate again. The operator has no sign-off duty (D4).

The private repository's open issues go through the relevance pass, and the survivors move to the companion or are filed again in public with the private details removed ([appendix I](I-issues-and-projects.md#importing-the-survivors)). The repository is archived only when none are open.

### What public CI proves for a private-origin import

Public CI can't read a private original, so the byte comparison happens where the original is readable, and public CI checks a manifest.

- **On the host that builds the import.** The tool writes a manifest into the import commit: each public file's new path and SHA-256 hash, and the source commit's hash. It compares each public file with the original byte for byte and puts the result and the manifest's digest in the scan report.
- **In the companion's CI.** A workflow on a private runner checks out the original at the frozen commit and the candidate from private staging, and repeats the comparison before the public push. It checks the public pull request's exact head again for approval. It also checks that every source file is in the manifest or in the held-back list, and that list never leaves the companion.
- **In public CI.** The `import` job checks that each file the commit adds matches its manifest hash or is listed as rewritten or generated, and that no executable reference to an old repository remains. The private report and its verdicts stay in the companion.
- **At approval.** An import is tier 2, and both review runs have the companion. Each reads the companion's check for the exact head and clears the head only when it's green, so the reviews App never approves an import whose private comparison failed or didn't run.

Public CI therefore proves only that the commit matches its own manifest. The match to the private bytes is proven in the companion, and it reaches the merge through the approval, never through public CI. The two pre-push verdicts in the companion bind the reviewed commit and report.

## What becomes public that isn't today

- Every future change to ops playbooks, agent configuration, and public memories, with its review discussion.
- The pace and shape of the factory's work, in one commit log.
- CI logs for all of the above.
