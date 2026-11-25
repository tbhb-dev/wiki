---
type: Design Appendix
title: One public monorepo for the factory layout and ownership
description: The monorepo's tree, where each repository lands, the dependency and visibility rules, the approval rules, the tier 2 path map, and how Apps and rulesets map.
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
MODULE.bazel  go.mod  pyproject.toml  pnpm-workspace.yaml  renovate.json5
AGENTS.md           agent instructions, and the tier 2 path map
BUILD.bazel         root targets, including the layering test
.github/            workflows and rulesets
factory/
  tbhbdev/          the command-line tool
  rules/            renovate-config/      runner-smoke/
  agent-config/     agent-memories/       agent-capacity/
  event-gateway/    telemetry/            dashboard/
  pm/               sm/                   work/
  evals/            ops/                  dotfiles/
products/           <name>/, empty today
lib/                design/, and shared code as it appears
docs/               wiki/  research/  spikes/
third_party/        vendored code, patches, and the private stub module
```

`factory/` is one boundary and has no levels inside it (D14). Every factory project is one folder directly under `factory/`, and the path says nothing about what kind of project it is. The boundary is what the build enforces: a product depends on `lib/` only, and the factory never depends on a product.

A project is a direct child of `factory/`, `products/`, `lib/`, or `docs/`. Each project folder has a `README.md` that says what the project does. No per-folder file names an owner, and no folder has an `OWNERS` file (D30).

Two placements differ from the starting sketch in the brief:

- **`lib/design`, not `docs/design`.** The design repository is a workspace of published packages, not prose, and products need to depend on it (D19).
- **The stub module in `third_party/`.** [Appendix D](D-public-private.md) explains it.

Go, Python, and TypeScript projects sit side by side under `factory/`, and all of them read the shared dependency roots in [appendix H](H-dependencies-and-renovate.md).

## Where each repository goes

Every repository arrives as one copy commit with no history (D10). One source repository becomes one folder. The last column says what the copy leaves out.

| Repository | Path | Left out of the public copy |
| --- | --- | --- |
| renovate-config | `factory/renovate-config` | root configuration |
| rules | `factory/rules` | root configuration |
| runner-smoke | `factory/runner-smoke` | root configuration |
| tools | `factory/tbhbdev` (D23) | root configuration |
| agent-config | `factory/agent-config` | the operator-account folder |
| agent-memories | `factory/agent-memories` | every memory not marked public |
| agent-capacity | `factory/agent-capacity` | root configuration |
| event-gateway, telemetry, dashboard | `factory/<name>` | root configuration |
| pm, sm, work | `factory/<name>` | root configuration, and what sm's scan holds back. Nobody read work in this pass |
| evals | `factory/evals` | root configuration |
| ops | `factory/ops` | inventory, host values, and variable files |
| dotfiles | `factory/dotfiles` | root configuration |
| design | `lib/design` | root configuration |
| wiki, research, spikes | `docs/<name>` | root configuration |
| github-metrics, graph, bazel-r2-cache | `factory/<name>` | root configuration, if D1 is yes |
| go.tbhb.dev | `factory/go-vanity` | root configuration, if D1 is yes |

"Root configuration" means each repository's own module file, lock files, lint settings, Renovate file, and workflows, which the shared roots and the root workflows replace. awrel, gomodscan, the Ansible collections, and the four open source tools stay standalone under D1's recommendation. The private counterparts of wiki, research, spikes, evals, and tools go to the companion at the mirrored paths.

## Dependency rules in Bazel

| From | May depend on | Never on |
| --- | --- | --- |
| `products/<name>` | itself, `lib/`, and published modules | `factory/`, `third_party/`, another product |
| `factory/<project>` | itself, `lib/`, `third_party/`, published modules, and targets another factory project exports | `products/` |
| `lib/<name>` | `lib/`, `third_party/`, and published modules | `factory/`, `products/` |
| `docs/` | `lib/`, the rules | `products/`, factory internals |

A published module is one that resolves through a shared dependency root at a version: a Bazel module, or a package from a language's registry. The build rules count, because every consumer loads them as `@tbhb_rules` labels. Code vendored under `third_party/` isn't published, so a product reaches it only through a `lib/` project that wraps it.

Two mechanisms enforce the table.

**Visibility defaults.** Bazel targets are private to their package unless a build file says otherwise. A gazelle directive at the top of each project sets what gazelle writes for everything below it:

```starlark
# factory/event-gateway/BUILD.bazel
# gazelle:default_visibility //factory/event-gateway:__subpackages__

# lib/BUILD.bazel
# gazelle:default_visibility //visibility:public

# products/<name>/BUILD.bazel
# gazelle:default_visibility //products/<name>:__subpackages__
```

A factory project exports a target to other factory projects by naming `//factory:__subpackages__` on that target by hand. Go's own `internal/` rule still applies below each project.

**A layering test.** Anyone can widen a visibility line, so one test in the root `BUILD.bazel` runs five queries and fails when any returns a result:

```sh
bazel query 'somepath(//products/..., //factory/...)'
bazel query 'somepath(//factory/..., //products/...)'
bazel query 'somepath(//lib/..., //factory/... + //products/...)'
bazel query 'somepath(//docs/..., //products/...)'
bazel query 'deps(//products/..., 1) intersect //third_party/...'
```

The rules never appear in these queries, because they're separate modules with `@tbhb_rules` labels. That's what lets a product use them and still leave the monorepo later. [Appendix H](H-dependencies-and-renovate.md#extracting-a-product) has the extraction steps, and the first rule above is what keeps them short.

## Approval is separation of duties

No file in the monorepo lists a person, an account, or a team as the owner or approver of a path (D30). Any agent may change any path. Approval comes from five rules about the duties in one change:

1. **The author never approves their own change.**
2. **The reviewer is independent of the author.** A review is a fresh run with its own context, and for an agent-built change it uses a provider other than the builder's.
3. **A verdict covers only the exact head it judged.** A new commit needs a new verdict.
4. **Review weight follows what the changed paths can break.** The review tier comes from the changed paths and content, and the heaviest tier among them applies to the whole change.
5. **The merge needs the checks and the required independent approval.** Neither one is enough alone.

These rules replace what repository boundaries did, so the monorepo has no `CODEOWNERS` file, no `OWNERS` files, and no table of roles. The review tool reads the review policy and the tier 2 path map from the base branch, as it reads review policy today (existing rule). A pull request therefore can't change the rules that judge it.

## Review tiers

The tiers are the three in the wiki's first decision record, and this design adds none and changes none (existing, D31):

| Tier | Review | Applies to |
| --- | --- | --- |
| 0 | CI only | Prose-only changes in the wiki, research, and spikes, except design documents and decision records |
| 1 | One fresh run | The default for code, tests, refactors, documentation in code, and wiki design documents |
| 2 | Two fresh runs, with different variants and providers other than the builder's | Credentials, rulesets, permissions, agent harness configuration, deployments, the review and merge path, decision records, and destructive deletion |

The diff's paths and content set the tier. A reviewer may raise it, and it never falls. The record's repository words carry over as paths: the wiki, research, and spikes are `docs/wiki`, `docs/research`, and `docs/spikes`. Its rule for a first-party pin change has nothing left to cover inside the monorepo, and routine third-party updates still need only CI.

### The tier 2 path map

The rules repository's `AGENTS.md` has a "Tier 2 review paths" table: one row for each glob, with a class and a reason (existing). Class 2 protects the agent account boundary, and class 3 protects review, approval, and merge requirements. The monorepo's root `AGENTS.md` carries the monorepo's equivalent, in the same form (new).

The import of rules rewrites that table's rows to their new paths. A row for a file inside rules gains the `factory/rules/` prefix, and a row for a root file names the monorepo's own root file. These rows show the pattern:

| Row in rules today | Glob in the monorepo | Class |
| --- | --- | --- |
| `.claude/settings.json`, `.codex/rules/pull-request.rules` | The same paths at the monorepo root | 2 |
| `lint/private/commit_identity/**`, `lint/private/claude_settings/**` | `factory/rules/lint/private/commit_identity/**`, `factory/rules/lint/private/claude_settings/**` | 2 |
| `.github/workflows/renovate.yaml`, `.github/renovate.json5` | The root Renovate workflow and the root `renovate.json5` | 2 |
| `repo/baseline/files/**` rows | `factory/rules/repo/baseline/files/**`, for the same files | 2 or 3, as today |
| `AGENTS.md`, `CLAUDE.md`, for text that sets the review tier | The same files at the monorepo root | 3 |
| `.github/rulesets/**`, `.github/workflows/ci.yml` | The same paths at the monorepo root | 3 |
| `lint/repo_checks.*`, `lint/private/required_status_checks/**`, `lint/private/review_policy/**`, `lint/private/pr_size*` | The same paths under `factory/rules/` | 3 |
| `MODULE.bazel`, for the block that pins the review policy's source | Expected to need no row, because that source is in the tree at `factory/tbhbdev` and has its own rows | none |

Each later import adds rows for its own class 2 and class 3 paths, such as the review and merge code under `factory/tbhbdev` and the harness configuration under `factory/agent-config`. The map lists paths only. The decision record's content triggers, such as a credential, a deployment, or a destructive deletion, set tier 2 wherever they appear, with or without a row.

The map is separate from area labels. It says how heavily a change is reviewed, and an area label says which project an issue belongs to ([appendix I](I-issues-and-projects.md#area-labels)). Neither is derived from the other.

Text in any other file, including a nested `AGENTS.md`, can't lower a tier the map sets, which is the rule the rules repository already states (existing).

### The tier 0 workflow

The tier 0 workflow becomes one root workflow, written inline. Its auto-approval list is the prose paths under `docs/wiki`, `docs/research`, and `docs/spikes`, without `docs/wiki/designs` and `docs/wiki/adrs`. Its denial list is every glob in the tier 2 path map. It reads the changed paths from Git, not from a workflow path filter.

## Mapping Apps, rulesets, and branch rules

| Today, per repository | In the monorepo |
| --- | --- |
| An App is installed on chosen repositories, and its permissions cover each one whole | Each App is installed once and reaches every path (D2). The count of installations to audit drops from one per repository to one |
| A main ruleset per repository, nearly the same in each | One checked-in ruleset: no deletion, no force push, required signatures, merge commits only, resolved threads, the required `conclusion` check pinned to GitHub Actions, the independent-approval requirement below, and the merge queue rule |
| Some repositories have stricter settings than others | The single ruleset takes the strictest value of each setting. Plan step 4 reads every live ruleset and lists any setting that only one repository has |
| `CODEOWNERS` per repository | None. No file names an owner or an approver |
| Environments per repository, such as the main-only `cache` environment | The same environments once, with the same branch policies. Deploy environments move to the companion with the deploys |
| Required reviewers by role or by code owner | None. The approval rules above apply to every path alike, and the tier sets the weight |

The ruleset and the review tool's App enforce the independent approval together (new), and each part is a duty:

- **Not the author.** The ruleset requires one approving review, and GitHub doesn't count an author's approval of their own pull request (per docs). The ruleset also requires that someone other than the last pusher approves.
- **The exact head.** The ruleset dismisses an approval when a new commit arrives, so an approval always names the head it judged.
- **The tier's weight.** The review tool's App posts the approval only after the fresh runs the tier requires have cleared that head: one at tier 1, and two at tier 2 with different variants and providers other than the builder's. At tier 0, the tier 0 workflow posts it once `conclusion` is green.
- **Independence.** The review tool starts each review as a fresh run and refuses a reviewer whose provider is the builder's. It reads the builder's provider from the commits' trailers.

Which App is installed where today is unverified, because this pass couldn't read live installations. Plan step 4 lists them before the monorepo's Apps are installed.

The companion gets the same ruleset without the queue rule, with a required up-to-date branch. The same approval rules and the same tiers apply to it, read from the monorepo commit it pins. It can also restrict pushes by file path, which GitHub offers on private repositories (per docs).
