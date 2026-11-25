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
| ops | Ansible roles and playbooks, service definitions, dashboards, the ops packages, OpenTofu modules, and the schema for inventory and values | The inventory, group and host variables, OpenTofu variable files and backend settings, and the overlay network's configuration |
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
| Publish private-origin code | `tbhb` signs off each import's scan report, if D4 is yes | Holds the report and the sign-off |

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
| The same scan in CI | A job on each pull request and each queue candidate, reading the list from a secret (D6) | A commit from a host whose hook didn't run | new |
| Import scan and sign-off | The import tool, on a host with the list, before the first push (D4) | Private content in a private-origin repository | new |
| Review | The reviewer of each pull request | What a pattern can't catch | existing |

The private-name list lives in the companion, because the list is itself private. A host without the companion can't run the name scan. The hook installer therefore refuses to enable pushes to the monorepo from such a host when the host also holds a private checkout.

Hooks come from `tbhbdev alpha install` only, as today (existing rule). The push hook is new, and it scans every commit that the push would publish.

## Importing a private-origin repository

1. In the private repository, a pull request adds a path map: which globs are public and which go to the companion. It gets its review there, before anything is copied.
2. The import tool builds the public tree from the map, and the companion tree from the rest.
3. It runs gitleaks and the name scan over the public tree, and writes a report: every file, every finding, and every file held back.
4. The operator signs off the report (D4).
5. The tool commits the public tree as one signed copy commit, as for every import, and the companion tree as one commit in the companion. No history comes along, so nothing that was removed in the past is published.

The private repository's open issues go through the relevance pass, and the survivors move to the companion or are filed again in public with the private details removed ([appendix I](I-issues-and-projects.md#importing-the-survivors)). The repository is archived only when none are open.

## What becomes public that isn't today

- Every future change to ops roles, agent configuration, and public memories, with its review discussion.
- The pace and shape of the factory's work, in one commit log.
- CI logs for all of the above.

That's the point of the move, and it's also why D4 puts the operator in front of each first publication.
