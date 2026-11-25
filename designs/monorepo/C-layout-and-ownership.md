---
type: Design Appendix
title: One public monorepo for the factory layout and ownership
description: The monorepo's tree, where each repository lands, the dependency and visibility rules, OWNERS files, review tiers per path, and how Apps and rulesets map.
tags: [monorepo, layout, ownership, bazel, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo somepath subpackages tbhbdev gomodscan evals bzlmod -->

# One public monorepo for the factory layout and ownership

Back to [the design](README.md).

## The tree

```text
MODULE.bazel  go.mod  pyproject.toml  pnpm-workspace.yaml  OWNERS  renovate.json5
.github/            workflows, rulesets, the generated CODEOWNERS
factory/
  tbhbdev/          the command-line tool, here or at cli/tbhbdev/ (D23)
  agents/           config/  memories/  capacity/
  build/            rules/  renovate/  policy/  remote-cache/  runner-smoke/
  services/         event-gateway/  telemetry/  github-metrics/  graph/  go-vanity/
  apps/             dashboard/
  work/             pm/  sm/  work/
  evals/
  infra/            ops/  dotfiles/
products/           <name>/, empty today
lib/                design/, and shared code as it appears
docs/               wiki/  research/  spikes/
third_party/        vendored code, patches, and the private stub module
```

This design writes the tool's folder as `<cli>`. It's `factory/tbhbdev` or `factory/cli/tbhbdev`, and D23 picks one. The tree follows the coordinator's sketch, with four changes:

- **`lib/design`, not `docs/design`.** The design repository is a workspace of published packages, not prose, and products need to depend on it (D19).
- **`factory/apps/`.** The dashboard is a pair of Apple apps, which is neither a service nor a command-line tool.
- **`factory/build/policy/`.** The tier policy, the roles table, and the layering test need a home that the monorepo maintainer owns.
- **The stub module in `third_party/`.** [Appendix D](D-public-private.md) explains it.

A folder is a project root when its `OWNERS` file says `project: true`. Folder names say what a thing does. Go, Python, and TypeScript projects sit side by side under the same role folder, and all of them read the shared dependency roots in [appendix H](H-dependencies-and-renovate.md).

## Where each repository goes

Every repository arrives as one copy commit with no history (D10). The last column says what the copy leaves out.

| Repository | Path | Left out of the public copy |
| --- | --- | --- |
| renovate-config | `factory/build/renovate` | root configuration |
| rules | `factory/build/rules` | root configuration |
| runner-smoke | `factory/build/runner-smoke` | root configuration |
| tools | `<cli>` | root configuration |
| agent-config | `factory/agents/config` | the operator-account folder |
| agent-memories | `factory/agents/memories` | every memory not marked public |
| agent-capacity | `factory/agents/capacity` | root configuration |
| event-gateway, telemetry | `factory/services/<name>` | root configuration |
| dashboard | `factory/apps/dashboard` | root configuration |
| pm, sm, work | `factory/work/<name>` | root configuration, and what sm's scan holds back. Nobody read work in this pass |
| evals | `factory/evals` | root configuration |
| ops | `factory/infra/ops` | inventory, host values, and variable files |
| dotfiles | `factory/infra/dotfiles` | root configuration |
| design | `lib/design` | root configuration |
| wiki, research, spikes | `docs/<name>` | root configuration |
| github-metrics, graph | `factory/services/<name>` | root configuration, if D1 is yes |
| bazel-r2-cache | `factory/build/remote-cache` | root configuration, if D1 is yes |
| go.tbhb.dev | `factory/services/go-vanity` | root configuration, if D1 is yes |

"Root configuration" means each repository's own module file, lock files, lint settings, Renovate file, and workflows, which the shared roots and the root workflows replace. awrel, gomodscan, the Ansible collections, and the four open source tools stay standalone under D1's recommendation. The private counterparts of wiki, research, spikes, evals, and tools go to the companion at the mirrored paths.

## Dependency rules in Bazel

| From | May depend on | Never on |
| --- | --- | --- |
| `products/<name>` | itself, `lib/`, and published modules | `factory/`, `third_party/`, another product |
| `factory/<area>/<project>` | itself, `lib/`, `third_party/`, published modules, and targets another factory project exports | `products/` |
| `lib/<name>` | `lib/`, `third_party/`, and published modules | `factory/`, `products/` |
| `docs/` | `lib/`, the rules | `products/`, factory internals |

A published module is one that resolves through a shared dependency root at a version: a Bazel module, or a package from a language's registry. The build rules count, because every consumer loads them as `@tbhb_rules` labels. Code vendored under `third_party/` isn't published, so a product reaches it only through a `lib/` project that wraps it.

Two mechanisms enforce the table.

**Visibility defaults.** Bazel targets are private to their package unless a build file says otherwise. A gazelle directive at the top of each tree sets what gazelle writes for everything below it:

```starlark
# factory/services/event-gateway/BUILD.bazel
# gazelle:default_visibility //factory/services/event-gateway:__subpackages__

# lib/BUILD.bazel
# gazelle:default_visibility //visibility:public

# products/<name>/BUILD.bazel
# gazelle:default_visibility //products/<name>:__subpackages__
```

A factory project exports a target to other factory projects by naming `//factory:__subpackages__` on that target by hand. Go's own `internal/` rule still applies below each project.

**A layering test.** Anyone can widen a visibility line, so one test in `factory/build/policy` runs five queries and fails when any returns a result:

```sh
bazel query 'somepath(//products/..., //factory/...)'
bazel query 'somepath(//factory/..., //products/...)'
bazel query 'somepath(//lib/..., //factory/... + //products/...)'
bazel query 'somepath(//docs/..., //products/...)'
bazel query 'deps(//products/..., 1) intersect //third_party/...'
```

The rules never appear in these queries, because they're separate modules with `@tbhb_rules` labels. That's what lets a product use them and still leave the monorepo later. [Appendix H](H-dependencies-and-renovate.md#extracting-a-product) has the extraction steps, and the first rule above is what keeps them short.

## `OWNERS` files

Every project root has an `OWNERS` file, which includes each `products/<name>` and each `lib/<name>`, and so does each top-level folder. The nearest one up the tree from a changed file applies. The scaffolding creates the file for every path in the table above, before any code arrives.

```yaml
# <cli>/OWNERS, with example paths
role: tools-maintainer
project: true
tier: 1
paths:
  "src/tbhbdev/alpha/review/**": 2
  "src/tbhbdev/alpha/merge/**": 2
```

- `role` names a durable role from `factory/build/policy/roles.yaml`, which maps each role to its GitHub team.
- `tier` is the default review tier for the folder, and `paths` raises or lowers it for globs below.
- A generator writes `.github/CODEOWNERS` and the `area/<path>` issue labels from these files, and a test fails when either is stale ([appendix I](I-issues-and-projects.md#area-labels)).
- A project adds an `OWNERS` file to a subfolder when that part needs its own owner or its own issue area. Raising a tier needs only a `paths` entry.

The review tool reads `OWNERS` and the root policy from the base branch, as it reads review policy today (existing rule). A pull request can't change the rules that judge it.

## Review tiers by path

The tiers themselves don't change: tier 0 is CI only, tier 1 is one fresh review run, and tier 2 is two. A change takes the highest tier among its paths. `factory/build/policy/tiers.yaml` holds the floor, and no `OWNERS` file can go below it.

| Path | Owning role | Tier |
| --- | --- | --- |
| `.github/**`, rulesets, `OWNERS`, `factory/build/policy/**`, `.mcp.json`, agent instruction files | Monorepo maintainer | 2, floor |
| `factory/agents/config/**` | Agent-config maintainer | 2, floor |
| Review and merge code under `<cli>` | Tools maintainer | 2, floor |
| Deploy definitions and credentials handling under `factory/infra/ops` and each service | Ops maintainer | 2, floor |
| `docs/wiki/adrs/**` | Wiki maintainer | 2, floor |
| `docs/wiki/designs/**` | Wiki maintainer | 1 |
| Other prose under `docs/wiki`, `docs/research`, `docs/spikes` | Each folder's maintainer | 0 |
| `factory/build/rules/**`, `factory/build/renovate/**` | Rules maintainer | 1 |
| Everything else under `factory/`, `lib/`, `products/` | The project's role | 1 |
| Third-party version updates from Renovate | The monorepo maintainer, who owns the root lock files | 0 |

The tier record's rule that a first-party pin change gets one review run has nothing left to cover inside the monorepo. D8 asks the operator to amend that record so tiers key on paths alone.

The tier 0 workflow becomes one root workflow, written inline. Its auto-approval list is the three prose rows above, and its denial list is every floor path. It reads the changed paths from Git, not from a workflow path filter.

## Mapping Apps, rulesets, and branch rules

| Today, per repository | In the monorepo |
| --- | --- |
| An App is installed on chosen repositories, and its permissions cover each one whole | Each App is installed once and reaches every path (D2). The count of installations to audit drops from one per repository to one |
| A main ruleset per repository, nearly the same in each | One checked-in ruleset: no deletion, no force push, required signatures, a pull request with one approval, stale dismissal, last-push approval, resolved threads, merge commits only, two required checks, and the merge queue rule. The checks are `conclusion`, pinned to GitHub Actions, and `owners`, pinned to the review tool's App |
| Some repositories have stricter settings than others | The single ruleset takes the strictest value of each setting. Plan step 4 reads every live ruleset and lists any setting that only one repository has |
| `CODEOWNERS` per repository | One generated `CODEOWNERS` |
| Environments per repository, such as the main-only `cache` environment | The same environments once, with the same branch policies. Deploy environments move to the companion with the deploys |
| Per-repository required reviewers by role | The review tool's owner routing, plus the required `owners` check. The review tool's App sets it on each pull request head and each queue candidate, and it fails when the owning role of a changed path hasn't approved. The tier 0 workflow's approval satisfies it for a tier 0 change |

No Actions job feeds `owners`, because a workflow in a pull request could change its own judge, and the App reads `OWNERS` from the base branch. Whether GitHub's queue accepts an App's check on a candidate is unverified, and plan step 2 tests it.

Which App is installed where today is unverified, because this pass couldn't read live installations. Plan step 4 lists them before the monorepo's Apps are installed.

The companion gets the same ruleset without the queue rule, with a required up-to-date branch. It has no `OWNERS` files of its own, and the review tool routes a companion change by the public `OWNERS` file at the mirrored path. It can also restrict pushes by file path, which GitHub offers on private repositories (per docs).
