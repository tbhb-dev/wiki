---
type: Design Appendix
title: One public monorepo for the factory layout and ownership
description: The monorepo's tree, where each repository lands, the OWNERS files, the dependency and visibility rules, which identity or App carries out each duty, the review tiers, and how Apps and rulesets map.
tags: [monorepo, layout, ownership, bazel, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo somepath subpackages tbhbdev gomodscan evals bzlmod noimplicit -->

# One public monorepo for the factory layout and ownership

Back to [the design](README.md).

## The tree

```text
MODULE.bazel  go.mod  pyproject.toml  pnpm-workspace.yaml  renovate.json5
AGENTS.md           agent instructions, and the tier 2 path map
BUILD.bazel         root targets, including the layering test
.github/            workflows and rulesets
factory/
  cli/              the command-line tool, with its OWNERS file and its docs/
  agent-config/     agent-memories/       agent-capacity/
  event-gateway/    telemetry/            dashboard/
  pm/               sm/                   work/
  evals/            dotfiles/
  github-metrics/   bazel-r2-cache/       go-vanity/
ops/
  ansible/collections/
  runner-smoke/
products/           <name>/, empty today
lib/                design/  rules/  renovate-config/, and shared code as it appears
docs/               wiki/  research/  spikes/, and no project's own records
third_party/        vendored code, patches, and the private stub module
```

`factory/` is flat (D14). Each factory project has one folder directly under `factory/`. Bazel limits products to dependencies in `lib/` and confines factory targets to `factory/`. Factory projects must stay independent of products. Reusable GitHub workflows stay in `.github/workflows/` as repository infrastructure outside factory projects.

An `OWNERS` file marks a project under `factory/`, `products/`, `lib/`, or `docs/`, and at `ops/` or any folder below it (D36). Ops, runner-smoke, and the Ansible collections each have their own project folder. Each project folder has a `README.md`, an `OWNERS` file, and its own `docs/` folder. [OWNERS files](#owners-files) and [the top-level docs folder](#what-the-top-level-docs-folder-holds) describe the last two.

The starting sketch had different locations for these projects and the stub.

- **`lib/design`, not `docs/design`.** The design repository is a workspace of published packages, not prose, and products need to depend on it (D19).
- **The build rules in `lib/rules`.** Products and standalone repositories load the nested rules modules (D15 and D35).
- **The stub module in `third_party/`.** [Appendix D](D-public-private.md) explains it.

Go, Python, and TypeScript projects sit side by side under `factory/`, and all of them read the shared dependency roots in [appendix H](H-dependencies-and-renovate.md).

### OWNERS files

Every project folder has one `OWNERS` file, and it does two things (new, D30):

- **It names the owners.** An owner is an identity, and tbhb-dev has two: `tbhb`, the operator's account, and `tbhb-agent`, the account every agent run works under. The file lists one or both. It never names a job title, a team, or a kind of run.
- **It describes the project.** One line says what the project is, and a list names the projects it depends on.

```text
# factory/event-gateway/OWNERS
owners: tbhb, tbhb-agent
about: receives GitHub webhooks and hands them to the factory's services
depends-on: factory/telemetry
```

The file's exact format is this design's proposal, and plan step 4 settles it with the test that reads it. That test fails for a project folder with no `OWNERS` file, for an owner that isn't one of the two identities, and for a `depends-on` list that differs from the project's Bazel dependencies on other projects.

An owner answers for the project: a question that a diff can't settle goes to an identity the file names. Being an owner grants no approval and blocks nobody. Any agent run may change any path, and the duties in [the section below](#separation-of-duties) decide what merges. The monorepo has no `CODEOWNERS` file, because GitHub turns its entries into required reviewers, which would be a second approval rule.

### What the top-level docs folder holds

A project's needs, requirements, and knowledge articles live in the project's own `docs/` folder, not in the top-level `docs/` (D33):

```text
factory/cli/docs/needs/
factory/cli/docs/requirements/
factory/cli/docs/kb/
```

The top-level `docs/` keeps prose that belongs to no single project: the wiki's designs, decision records, runbooks, and model notes, plus research and spikes. Those three are projects too, so the wiki's own records sit at `docs/wiki/docs/needs`, `docs/wiki/docs/requirements`, and `docs/wiki/docs/kb`.

A project's `docs/` folder is part of the project. It isn't a project itself, so it gets no area label, no `OWNERS` file, and no coverage figure of its own. A pull request that changes only Markdown files in those three folders takes the lighter CI lane for the projects covered by [appendix F](F-ci.md#the-record-lane).

Each import's path map puts a source repository's needs, requirements, and knowledge articles in its project's three folders. Which repositories hold such records today wasn't read in this pass (unverified).

## Where each repository goes

Every repository arrives as one copy commit with no history (D10). One source repository becomes one folder. The last column says what the copy leaves out.

| Repository | Path | Left out of the public copy |
| --- | --- | --- |
| renovate-config | `lib/renovate-config` | root configuration |
| rules | `lib/rules` | root configuration. Keep the nested module files in `lib/rules` and move reusable workflows to `.github/workflows/` |
| runner-smoke | `ops/runner-smoke` | root configuration |
| tools | `factory/cli` (D23) | root configuration |
| agent-config | `factory/agent-config` | the operator-account folder |
| agent-memories | `factory/agent-memories` | every memory not marked public |
| agent-capacity | `factory/agent-capacity` | root configuration |
| event-gateway, telemetry, dashboard | `factory/<name>` | root configuration |
| pm, sm, work | `factory/<name>` | root configuration, and what sm's scan holds back. Nobody read work in this pass |
| evals | `factory/evals` | root configuration |
| ops | `ops/` | inventory, host values, and variable files, which go in the companion's `ops/` |
| dotfiles | `factory/dotfiles` | root configuration |
| design | `lib/design` | root configuration |
| wiki, research, spikes | `docs/<name>` | root configuration |
| github-metrics, bazel-r2-cache | `factory/<name>` | root configuration |
| go.tbhb.dev | `factory/go-vanity` | root configuration |
| Ansible collections | `ops/ansible/collections` | root configuration |

"Root configuration" means each repository's own module file, lock files, lint settings, Renovate file, and workflows, which the shared roots and the root workflows replace. gomodscan stays standalone, and awrel stays out because it will retire soon (D1). The four open source tools and the graph repository are out of scope and don't move (D11). The private counterparts of wiki, research, spikes, evals, and tools go to the companion at the mirrored paths.

## Dependency rules in Bazel

| From | May depend on | Never on |
| --- | --- | --- |
| `products/<name>` | itself and `lib/`, built with the rules | `factory/`, `ops/`, `third_party/`, another product, and a published module named directly |
| `factory/<project>` | itself, `lib/`, `third_party/`, published modules, and targets another factory project exports | `products/`, `ops/` |
| `lib/<name>` | `lib/`, `third_party/`, and published modules | `factory/`, `products/`, `ops/` |
| `docs/` | `lib/`, the rules | `products/`, `factory/`, `ops/` |
| `ops/` | Nothing (recommendation) | All Bazel targets (recommendation) |

No Bazel target may depend on `ops/`. The recommendation is for `ops/` to have no Bazel dependencies and run outside Bazel. Deployment tools can use the Ansible collections, private values from the companion's `ops/`, and built artifacts such as service images.

A published module is one that resolves through a shared dependency root at a version: a Bazel module, or a package from a language's registry. The factory and `lib/` may declare dependencies on published modules. Products use published modules and vendored code through `lib/` (D35). Product build files load the nested build rules in `lib/rules` as `@tbhb_rules` labels. The factory publishes nothing for use outside `factory/`. Anything products need goes in `lib/`. Code used by standalone repositories also goes there.

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

A factory project exports a target to other factory projects by naming `//factory:__subpackages__` on that target by hand. Every `factory/` target's Bazel visibility stays inside `factory/`, including for targets with no current consumers. Go's own `internal/` rule still applies below each project.

**A layering test.** Anyone can widen a visibility line, so one test in the root `BUILD.bazel` runs these queries and fails when any returns a result:

```sh
bazel query 'somepath(//products/..., //factory/...)'
bazel query 'somepath(//factory/..., //products/...)'
bazel query 'somepath(//lib/..., //factory/... + //products/...)'
bazel query 'somepath(//docs/..., //products/...)'
bazel query --noimplicit_deps 'deps(//products/..., 1) except (//products/...:* + //lib/...:*)'
# once for each folder P under products/
bazel query 'deps(//products/P/...) intersect (//products/...:* except //products/P/...:*)'
```

The fifth query finds a product's direct dependency on anything but a product or `lib/`, which covers `third_party/` and every published module, and the `:*` patterns count source files as well as rule targets. It leaves out implicit dependencies, so the rules' own toolchains don't count, and whether that flag excludes exactly those is unverified until plan step 4 writes the test. The sixth query runs once for each product folder and finds a dependency from one product to another, whatever a visibility line says. A visibility query also runs for each package outside `factory/`, including every package in `products/` and `lib/`. For a package at `products/example`, it runs `bazel query 'visible(//products/example:*, //factory/...:*)'` and fails if any factory target is visible there. The test also rejects factory visibility declarations that grant access outside `factory/`, including public visibility when there are no outside targets yet.

If the recommendation to run `ops/` outside Bazel is adopted, the layering test also rejects Bazel packages under `ops/`.

The rules never appear in these queries, because they're separate modules with `@tbhb_rules` labels. That's what lets a product use them and still leave the monorepo later. [Appendix H](H-dependencies-and-renovate.md#extracting-a-product) has the extraction steps, and the first rule above is what keeps them short.

## Separation of duties

tbhb-dev keeps four duties apart in every change: build, review, approve, and merge (D30). An identity or an App carries out each one. There are two identities, `tbhb` and `tbhb-agent`, plus the GitHub Apps that the wiki's records and designs already name.

| Duty | Who carries it out | How |
| --- | --- | --- |
| Build | `tbhb-agent`, in an agent run. `tbhb` when the operator writes the change, and the Renovate App for a dependency update | The builder pushes the branch and opens the pull request, so the builder is the author |
| Review | `tbhb-agent`, in a fresh run that isn't the build run. The run that commissioned the build may start it (existing decision) | The run reads the exact head and publishes its verdict on the pull request through the reviews App |
| Approve | The reviews App at tiers 1 and 2, and the tier 0 workflow at tier 0. The approver App `tbhb-dev-renovate-approver` for a pin-only Renovate update. `tbhb` where an existing rule asks for the operator | Each posts a GitHub approval for one head, after the reviews that head's tier requires |
| Merge | `tbhb-agent` runs `tbhbdev alpha merge`, or `tbhb` does, and GitHub's merge queue lands the commit | The merge commit's `Merged-by` trailer records who merged (existing) |

`tbhb-agent` builds and reviews, so the account alone can't keep those two duties apart. The run does: a review run shares no context with the build run, and for an agent-built change it uses another provider. The reviews App refuses an approval that traces back to the opener's own run (existing decision).

Until the reviews App can post approvals, the existing `tbhb-approve` command posts them for the exact head (existing decision). It posts as the reviews App, `tbhb-dev-agent-reviews`, and not as `tbhb` (existing, [appendix A](A-evidence.md#rules-review-and-ci-today)).

Five rules hold across the duties:

1. **The author never approves their own change.**
2. **The reviewer is independent of the author.** A review is a fresh run with its own context, and for an agent-built change it uses a provider other than the builder's.
3. **A verdict covers only the exact head it judged.** A new commit needs a new verdict.
4. **Review weight follows what the changed paths can break.** The review tier comes from the changed paths and content, and the heaviest tier among them applies to the whole change.
5. **The merge needs the checks and the required independent approval.** Neither one is enough alone.

These rules replace what repository boundaries did. No path has its own approver, and the `OWNERS` files take no part in approval. The review tool reads the review policy and the tier 2 path map from the base branch, as it reads review policy today (existing rule). A pull request therefore can't change the rules that judge it.

### Review routing

No file routes a review to a name. `tbhb-agent` starts every review as a fresh run, and the changed paths set how many runs there are and which providers they use. A reviewer's question about a project's direction goes to an identity in that project's `OWNERS` file, and the answer is a comment, never an approval.

Two kinds of change wait for `tbhb` under rules that already exist. Neither depends on who owns a path:

- **A decision record change.** It needs the operator's approval after its two reviews (existing, the review tier record).
- **A Go port pull request for the front, dispatch, or plugin plumbing.** The operator reads it before merge (D29).

A private-origin import first needs two independent `tbhb-agent` reviews in a private workspace before any public push (D4). [Appendix D](D-public-private.md#importing-a-private-origin-repository) gives the gate.

## Review tiers

The tiers are the three in the wiki's first decision record, and this design adds none and changes none (existing, D31):

| Tier | Review | Who runs the reviews | Who approves | Applies to |
| --- | --- | --- | --- | --- |
| 0 | CI only | Nobody. No review run starts | The tier 0 workflow, once `conclusion` is green | Prose-only changes in the wiki, research, and spikes, except design documents and decision records |
| 1 | One fresh run | `tbhb-agent`, in one run that isn't the build run | The reviews App, after that run clears the head | The default for code, tests, refactors, documentation in code, and wiki design documents |
| 2 | Two fresh runs, with different variants and providers other than the builder's | `tbhb-agent`, in two runs that aren't the build run | The reviews App, after both runs clear the head. A decision record change also needs `tbhb` | Credentials, rulesets, permissions, agent harness configuration, deployments, the review and merge path, decision records, and destructive deletion |

At every tier the author is the builder, and the builder never approves. The wiki's tier 0 workflow receives the approver App's credentials today (existing, read from the wiki's workflow file), and the monorepo's tier 0 workflow keeps the identity it posts with.

The diff's paths and content set the tier. A reviewer may raise it, and it never falls. The record's repository words carry over as paths: the wiki, research, and spikes are `docs/wiki`, `docs/research`, and `docs/spikes`. Its rule for a first-party pin change has nothing left to cover inside the monorepo, and routine third-party updates still need only CI as review, with the approval that [appendix H](H-dependencies-and-renovate.md#approval-and-owner-notices) describes.

The record lane changes which checks CI runs, and it changes no tier (D33). A record pull request under `docs/wiki`, `docs/research`, or `docs/spikes` is prose there, so it's tier 0. One under `factory/`, `products/`, or `lib/` is documentation in code, so it's tier 1 and gets one fresh review run. A knowledge article that holds a credential, or anything else the content triggers name, is tier 2 like any other file.

### The tier 2 path map

The rules repository's `AGENTS.md` has a "Tier 2 review paths" table: one row for each glob, with a class and a reason (existing). Class 2 protects the agent account boundary, and class 3 protects review, approval, and merge requirements. The monorepo's root `AGENTS.md` carries the monorepo's equivalent, in the same form (new).

The import of rules rewrites that table's rows to their new paths. Prefix each path inside the imported rules project with `lib/rules/`. Root files use their paths relative to the monorepo root. These rows show the pattern:

| Row in rules today | Glob in the monorepo | Class |
| --- | --- | --- |
| `.claude/settings.json`, `.codex/rules/pull-request.rules` | The same paths at the monorepo root | 2 |
| `lint/private/commit_identity/**`, `lint/private/claude_settings/**` | `lib/rules/lint/private/commit_identity/**`, `lib/rules/lint/private/claude_settings/**` | 2 |
| `.github/workflows/renovate.yaml`, `.github/renovate.json5` | The root Renovate workflow and the root `renovate.json5` | 2 |
| `repo/baseline/files/**` rows | `lib/rules/repo/baseline/files/**`, for the same files | 2 or 3, as today |
| `AGENTS.md`, `CLAUDE.md`, for text that sets the review tier | The same files at the monorepo root | 3 |
| `.github/rulesets/**`, `.github/workflows/ci.yml` | The same paths at the monorepo root. `ci.yml` holds the record lane's path check, so this row covers it | 3 |
| none, because the rule is new | The record-check rule under `lib/rules`, at the path its pull request gives it | 3 |
| `lint/repo_checks.*`, `lint/private/required_status_checks/**`, `lint/private/review_policy/**`, `lint/private/pr_size*` | The same paths under `lib/rules/` | 3 |
| `MODULE.bazel`, for the block that pins the review policy's source | Expected to need no row, because that source is in the tree at `factory/cli` and has its own rows | none |

Each later import adds rows for its own class 2 and class 3 paths, such as the review and merge code under `factory/cli` and the harness configuration under `factory/agent-config`. The map lists paths only. The decision record's content triggers, such as a credential, a deployment, or a destructive deletion, set tier 2 wherever they appear, with or without a row.

The map is separate from area labels. It says how heavily a change is reviewed, and an area label says which project an issue belongs to ([appendix I](I-issues-and-projects.md#area-labels)). Neither is derived from the other.

Text in any other file, including a nested `AGENTS.md`, can't lower a tier the map sets, which is the rule the rules repository already states (existing).

### The tier 0 workflow

The tier 0 workflow becomes one root workflow, written inline. Its auto-approval list is the prose paths under `docs/wiki`, `docs/research`, and `docs/spikes`, without `docs/wiki/designs` and `docs/wiki/adrs`. Its denial list is every glob in the tier 2 path map. It reads the changed paths from Git, not from a workflow path filter. The list doesn't grow for the record lane: a project's `docs/` folder under `factory/`, `products/`, or `lib/` stays outside it.

## Mapping Apps, rulesets, and branch rules

| Today, per repository | In the monorepo |
| --- | --- |
| An App is installed on chosen repositories, and its permissions cover each one whole | Each App is installed once and reaches every path (D2). The count of installations to audit drops from one per repository to one |
| A main ruleset per repository, nearly the same in each | One checked-in ruleset: no deletion, no force push, required signatures, merge commits only, resolved threads, the required `conclusion` check pinned to GitHub Actions, the independent-approval requirement below, and the merge queue rule |
| Some repositories have stricter settings than others | The single ruleset takes the strictest value of each setting. Plan step 4 reads every live ruleset and lists any setting that only one repository has |
| `CODEOWNERS` per repository | None. Each project's `OWNERS` file names its owners, and no file names an approver |
| Environments per repository, such as the main-only `cache` environment | The same environments once, with the same branch policies. Deploy environments move to the companion with the deploys |
| Required reviewers by code owner | None. The duties above apply to every path alike, and the tier sets the weight |

The ruleset and the reviews App enforce the independent approval together (new):

- **Not the author.** The ruleset requires one approving review, and GitHub doesn't count an author's approval of their own pull request (per docs). The ruleset also requires that someone other than the last pusher approves.
- **The exact head.** The ruleset dismisses an approval when a new commit arrives, so an approval always names the head it judged.
- **The tier's weight.** The reviews App posts the approval only after the fresh runs the tier requires have cleared that head: one at tier 1, and two at tier 2 with different variants and providers other than the builder's. At tier 0, the tier 0 workflow posts it once `conclusion` is green.
- **Independence.** The review tool starts each review as a fresh run and refuses a reviewer whose provider is the builder's. It reads the builder's provider from the commits' trailers.

The Apps this design relies on are the ones the wiki's records and designs already name: the reviews App, the approver App `tbhb-dev-renovate-approver`, the Renovate App, and the checks App `tbhb-dev-issue-checks`. The checks App writes issue check results and takes no part in a pull request's four duties. This design creates no App. Which App is installed where today is unverified, because this pass couldn't read live installations. Plan step 4 lists them before the monorepo's Apps are installed.

The companion gets the same ruleset without the queue rule, with a required up-to-date branch. The same duties, carried out by the same identities and Apps, and the same tiers apply to it, read from the monorepo commit it pins. It can also restrict pushes by file path, which GitHub offers on private repositories (per docs).
