---
type: Design Appendix
title: One public monorepo for the factory dependencies and Renovate
description: The one dependency root per language, the one-version rule and its exception process, how a product leaves the shared roots, and how one Renovate run keeps them current.
tags: [monorepo, dependencies, renovate, bazel, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo bzlmod tbhbdev rdeps automerge -->

# One public monorepo for the factory dependencies and Renovate

Back to [the design](README.md).

## One root per language

Each language has one manifest and one lock file at the repository root, and Bazel reads each lock (new, D18). A root appears with the first project that needs it.

| Language | Root files | How Bazel reads it |
| --- | --- | --- |
| Bazel modules | `MODULE.bazel`, `MODULE.bazel.lock` | Directly |
| Python | One uv workspace in `pyproject.toml`, and `uv.lock` | The `rules_python` pip hub |
| Go | One `go.mod` and `go.sum` | Gazelle's `go_deps` extension |
| Rust | One Cargo workspace, and `Cargo.lock` | The `rules_rust` crate universe |
| Node | One pnpm workspace, and `pnpm-lock.yaml` | `rules_js` |
| Swift | A root `Package.swift` and `Package.resolved`, for external packages only | `rules_swift_package_manager` |

One root gives each external dependency one version across the repository. An update is then one pull request that every user of the package is tested against, where today the same update lands in several repositories on different days (proven, [appendix A](A-evidence.md#change-counts)). The cost is that an upgrade has to work for every user at once, and the exception process below is the release valve.

Three details complete the table:

- **Go has no `go.work`.** The root module's path sits on the vanity domain, so the organization's name isn't in any import. Keeping each project's old module path was the one case for a workspace file, and the clean break rewrites those paths at import anyway.
- **Swift code inside the repository is plain `swift_library` targets.** Only external packages go through the root package file, and Swift builds need macOS runners ([appendix F](F-ci.md#runner-sizing)). Whether the dashboard's apps build under Bazel today wasn't read in this pass (unverified).
- **The nested rules modules declare dependencies too.** They stay loadable alone (D15), and a test fails when one declares a version that differs from the root's.

## A second version

A lock file resolves a second version of a dependency only through an entry in the root `version-exceptions.yaml`. An entry is a code change, so it takes tier 1 unless a reviewer raises it.

- An entry names the dependency, both versions, the project that needs the second, the reason, and an end date at most 90 days out.
- A test fails when any lock holds two versions of one package with no entry, and when an entry has passed its end date.
- The mechanism is each tool's own, such as `multiple_version_override` in Bazel and a named override in pnpm. Go treats a new major version as a different module, which needs no entry.
- A Go project that can't share a minor version is the one written case for `go.work`. Its entry says so, and the workspace file goes when the entry ends.

## Extracting a product

A product depends only on `lib/` and is built with the rules, so it can leave. `tbhbdev alpha monorepo extract products/<name>` is planned to do it in these steps:

1. Copy the folder into a new repository as one commit that names the monorepo commit. No history goes along, and the monorepo keeps it.
2. Ask Bazel which external dependencies the product's targets reach through `lib/`, and write new manifests that list only those, at the versions the shared locks hold.
3. Generate fresh lock files from those manifests, and fail if any resolved version differs from the shared lock's.
4. Replace each `lib/` dependency with a published version of that library. A `lib/` project that a product uses therefore needs a release first.
5. Load the rules by `git_override`, as any standalone repository does.

The extraction is done when the new repository builds and tests with no path into the monorepo. The product's folder is then deleted in one pull request, and Renovate drops whatever no other project uses.

## Renovate

Today a self-hosted Renovate runs once for each repository, from a reusable workflow, on a GitHub App token (existing). In the monorepo it runs once for the whole repository.

| Topic | Design |
| --- | --- |
| Internal pins | None exist inside the monorepo, so Renovate has none to move. The companion's one pin on a monorepo commit is the exception, and the companion's own run moves it |
| One version | The single root for each language is the enforcement. Renovate has no second manifest to update differently |
| Runs | One root workflow, written inline, on today's schedule and the same App token. The root `renovate.json5` loads the presets from `factory/renovate-config` in the same repository |
| Grouping | One group for each lock file holds its minor, patch, and digest updates, opened once a day. Each major version gets its own pull request, and a security update opens at once |
| Automerge | A green non-major update needs only CI under the review tier record, and it still needs an approval to enter the queue. [Approval and owner notices](#approval-and-owner-notices) says who gives it. Renovate's platform automerge then adds it to the merge queue, so it lands through the same queue as every change |
| Limits | At most 10 Renovate pull requests are open at once, which keeps the bot well inside the 1,000 open pull requests GitHub recommends for a branch |
| Failures | A grouped update waits on no one while it's green. When it fails, the workflow comments with each failing project, from the failing targets' paths, and notifies that project's owners. A major update's body lists the projects that depend on the package, from `bazel query 'rdeps(...)'`, and notifies theirs |

### Approval and owner notices

The tier 0 workflow never approves a dependency update, because its list holds prose paths only and no manifest or lock file ([appendix C](C-layout-and-ownership.md#the-tier-0-workflow)). A qualifying update gets its approval from the approver App, `tbhb-dev-renovate-approver`, under the [Renovate approval design](../renovate-auto-approval/README.md) (existing decision). The monorepo carries that design's workflow inline.

- **What qualifies.** The Renovate App is the author, the update is a patch, a minor, or a digest, the diff holds only the pin moves and lock entries that design's matcher accepts, and `conclusion` is green on the exact head.
- **What doesn't.** An update the matcher refuses takes tier 1: `tbhb-agent` starts one fresh review run, and the reviews App approves. A major update always takes that path.
- **The matcher's file list isn't widened here.** Which root manifests and locks it accepts is that design's list, and widening it is the operator's call there. Plan step 7 records how many updates qualify in the first week.
- **Owners get a notice.** The workflow reads the `OWNERS` file of each affected project and mentions the identities it names in its comment, so GitHub notifies them. The notice asks for no approval, and an owner's reply is a comment.

### Tasks after an upgrade

The Bazel preset runs a task after each upgrade that refreshes `MODULE.bazel.lock` (existing). With one lock for the whole repository, three things keep that cheap:

- **Tasks run once for each branch.** The preset sets `executionMode` to `branch`, so a group of 20 updates refreshes the lock once.
- **Few branches exist.** Every update to a shared lock conflicts with every other open update to it, and Renovate then rebases and reruns the tasks. One group for each lock means at most one open non-major branch for each lock, so that rebase happens once.
- **Tasks refresh locks and nothing else.** Version updates don't change which packages exist, so no task runs gazelle over the tree. The `repo` job in CI still fails a branch whose build files are stale.

The uv, Go, Cargo, and pnpm locks are refreshed by Renovate's own managers and need no task (per docs). How long one lock refresh takes at the monorepo's size is unverified, and plan step 7 records it in the first week of runs.
