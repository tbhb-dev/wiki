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

<!-- cspell:words monorepo somepath subpackages tbhbdev gomodscan evals -->

# One public monorepo for the factory layout and ownership

Back to [the design](README.md).

## The tree

```text
MODULE.bazel  go.work  pyproject.toml  pnpm-workspace.yaml  OWNERS  renovate.json5
.github/            workflows, rulesets, the generated CODEOWNERS
factory/
  cli/tbhbdev/      the command-line tool
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

The tree follows the coordinator's sketch, with four changes:

- **`lib/design`, not `docs/design`.** The design repository is a workspace of published packages, not prose, and products need to depend on it (D19).
- **`factory/apps/`.** The dashboard is a pair of Apple apps, which is neither a service nor a command-line tool.
- **`factory/build/policy/`.** The tier policy, the roles table, and the layering test need a home that the monorepo maintainer owns.
- **The stub module in `third_party/`.** [Appendix D](D-public-private.md) explains it.

A folder is a project when it has its own `OWNERS` file and its own language manifest. Folder names say what a thing does. Go, Python, and TypeScript projects sit side by side under the same role folder.

## Where each repository goes

| Repository | Path | Import |
| --- | --- | --- |
| rules | `factory/build/rules` | history |
| renovate-config | `factory/build/renovate` | history |
| runner-smoke | `factory/build/runner-smoke` | squash, if D5 is yes |
| tools | `factory/cli/tbhbdev` | history |
| agent-config | `factory/agents/config` | squash, public parts |
| agent-memories | `factory/agents/memories` | squash, public parts |
| agent-capacity | `factory/agents/capacity` | history |
| event-gateway | `factory/services/event-gateway` | history |
| telemetry | `factory/services/telemetry` | history |
| dashboard | `factory/apps/dashboard` | history |
| pm, sm, work | `factory/work/<name>` | history for pm, squash for sm, unread for work |
| evals | `factory/evals` | history |
| ops | `factory/infra/ops` | squash, roles and playbooks |
| dotfiles | `factory/infra/dotfiles` | squash, if D5 is yes |
| design | `lib/design` | history |
| wiki, research, spikes | `docs/<name>` | history |
| github-metrics, graph | `factory/services/<name>` | history, if D1 is yes |
| bazel-r2-cache | `factory/build/remote-cache` | history, if D1 is yes |
| go.tbhb.dev | `factory/services/go-vanity` | history, if D1 is yes |

awrel, gomodscan, the Ansible collections, and the four open source tools stay standalone under D1's recommendation. The private counterparts of wiki, research, spikes, evals, and tools go to the companion at the mirrored paths.

## Dependency rules in Bazel

| From | May depend on | Never on |
| --- | --- | --- |
| `products/<name>` | itself, `lib/`, `third_party/`, published modules | `factory/`, another product |
| `factory/<area>/<project>` | itself, `lib/`, `third_party/`, and targets another factory project exports | `products/` |
| `lib/<name>` | `lib/`, `third_party/` | `factory/`, `products/` |
| `docs/` | `lib/`, the rules | `products/`, factory internals |

Two mechanisms enforce the table.

**Visibility defaults.** Bazel targets are private to their package unless a build file says otherwise. A gazelle directive at the top of each tree sets what gazelle writes for everything below it:

```starlark
# factory/cli/tbhbdev/BUILD.bazel
# gazelle:default_visibility //factory/cli/tbhbdev:__subpackages__

# lib/BUILD.bazel
# gazelle:default_visibility //visibility:public

# products/<name>/BUILD.bazel
# gazelle:default_visibility //products/<name>:__subpackages__
```

A factory project exports a target to other factory projects by naming `//factory:__subpackages__` on that target by hand. Go's own `internal/` rule still applies inside each module.

**A layering test.** Anyone can widen a visibility line, so one test in `factory/build/policy` runs four queries and fails when any returns a path:

```sh
bazel query 'somepath(//products/..., //factory/...)'
bazel query 'somepath(//factory/..., //products/...)'
bazel query 'somepath(//lib/..., //factory/... + //products/...)'
bazel query 'somepath(//docs/..., //products/...)'
```

The rules never appear in these queries, because they're separate modules with `@tbhb_rules` labels. That's what lets a product use them and still leave the monorepo later.

To extract a product, `git subtree split` its folder, then replace each `lib/` dependency with a published version of that library. The first rule above is what keeps that list short.

## `OWNERS` files

Each project folder, and each top-level folder, has an `OWNERS` file. The nearest one up the tree from a changed file applies.

```yaml
# factory/cli/tbhbdev/OWNERS, with example paths
role: tools-maintainer
tier: 1
paths:
  "src/tbhbdev/alpha/review/**": 2
  "src/tbhbdev/alpha/merge/**": 2
```

- `role` names a durable role from `factory/build/policy/roles.yaml`, which maps each role to its GitHub team.
- `tier` is the default review tier for the folder, and `paths` raises or lowers it for globs below.
- A generator writes `.github/CODEOWNERS` from these files, and a test fails when it's stale. GitHub's interface then shows the same owners.

The review tool reads `OWNERS` and the root policy from the base branch, as it reads review policy today (existing rule). A pull request can't change the rules that judge it.

## Review tiers by path

The tiers themselves don't change: tier 0 is CI only, tier 1 is one fresh review run, and tier 2 is two. A change takes the highest tier among its paths. `factory/build/policy/tiers.yaml` holds the floor, and no `OWNERS` file can go below it.

| Path | Owning role | Tier |
| --- | --- | --- |
| `.github/**`, rulesets, `OWNERS`, `factory/build/policy/**`, `.mcp.json`, agent instruction files | Monorepo maintainer | 2, floor |
| `factory/agents/config/**` | Agent-config maintainer | 2, floor |
| Review and merge code under `factory/cli/tbhbdev` | Tools maintainer | 2, floor |
| Deploy definitions and credentials handling under `factory/infra/ops` and each service | Ops maintainer | 2, floor |
| `docs/wiki/adrs/**` | Wiki maintainer | 2, floor |
| `docs/wiki/designs/**` | Wiki maintainer | 1 |
| Other prose under `docs/wiki`, `docs/research`, `docs/spikes` | Each folder's maintainer | 0 |
| `factory/build/rules/**`, `factory/build/renovate/**` | Rules maintainer | 1 |
| Everything else under `factory/`, `lib/`, `products/` | The project's role | 1 |
| Third-party version updates from Renovate | The project's role | 0 |

The tier record's rule that a first-party pin change gets one review run has nothing left to cover inside the monorepo. D8 asks the operator to amend that record so tiers key on paths alone.

The tier 0 workflow becomes one root workflow. Its auto-approval list is the three prose rows above, and its denial list is every floor path.

## Mapping Apps, rulesets, and branch rules

| Today, per repository | In the monorepo |
| --- | --- |
| An App is installed on chosen repositories, and its permissions cover each one whole | Each App is installed once and reaches every path (D2). The count of installations to audit drops from one per repository to one |
| A main ruleset per repository, nearly the same in each | One checked-in ruleset: no deletion, no force push, required signatures, a pull request with one approval, stale dismissal, last-push approval, resolved threads, merge commits only, the `conclusion` check pinned to GitHub Actions, and the merge queue rule |
| Some repositories have stricter settings than others | The single ruleset takes the strictest value of each setting. Plan step 4 reads every live ruleset and lists any setting that only one repository has |
| `CODEOWNERS` per repository | One generated `CODEOWNERS` |
| Environments per repository, such as the main-only `cache` environment | The same environments once, with the same branch policies. Deploy environments move to the companion with the deploys |
| Per-repository required reviewers by role | The review tool's owner routing, plus a required `owners` check, set by the review tool's App, that fails when the owning role of a changed path hasn't approved |

Which App is installed where today is unverified, because this pass couldn't read live installations. Plan step 4 lists them before the monorepo's Apps are installed.

The companion gets the same ruleset without the queue rule, with a required up-to-date branch. It can also restrict pushes by file path, which GitHub offers on private repositories (per docs).
