---
type: Design Appendix
title: One public monorepo for the factory issues and Projects
description: Where issues live after the move, the two Projects, the area labels computed from the tree, the issue rules, the relevance pass over existing issues, and the gate before an old repository is archived.
tags: [monorepo, issues, projects, labels, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words monorepo tbhbdev -->

# One public monorepo for the factory issues and Projects

Back to [the design](README.md).

## Two repositories, two Projects

Issues live in the monorepo and in the companion, and both have Issues and Projects turned on (D28). An issue goes to the companion when it can't be stated without a private detail, and to the monorepo otherwise.

| Project | Visibility | Holds |
| --- | --- | --- |
| The roadmap Project | Public | The monorepo's public issues (D26) |
| The main private Project | Private | All issues from both repositories (D26) |

`tbhbdev alpha issue create` adds every new issue to the main private Project and each new monorepo issue to the public roadmap as well, and the sync check below reports any that a person filed by hand. The scaffolding creates both Projects in plan step 4.

## Area labels

Each issue carries exactly one area label (D28). A project is a folder under `factory/`, `products/`, `lib/`, or `docs/` that holds an `OWNERS` file (D36). The generator finds those folders in the monorepo's tree and names each label after the folder's path, so no list file exists and nothing inside an `OWNERS` file becomes a label.

| Label | One for each |
| --- | --- |
| `area/factory/<project>` | folder under `factory/` that holds an `OWNERS` file |
| `area/products/<name>` | folder under `products/` that holds an `OWNERS` file |
| `area/lib/<name>` | folder under `lib/` that holds an `OWNERS` file |
| `area/docs/<name>` | folder under `docs/` that holds an `OWNERS` file |
| `area/repo` | the repository, once: CI, `.github/`, rulesets, `MODULE.bazel`, and the language roots |
| `area/third_party` | the repository, once: everything under `third_party/` |
| `area/none` | the repository, once: an issue that belongs to no project |

An issue about the event gateway's retry logic is `area/factory/event-gateway`. An issue about the command-line tool is `area/factory/tbhbdev`.

A project's label covers the project's own records. An issue about a need, a requirement, or a knowledge article under `factory/tbhbdev/docs` is `area/factory/tbhbdev`, not `area/docs/wiki` (D33). The `area/docs/<name>` labels cover only the wiki, research, and spikes, and a project's `docs/` folder never gets a label of its own.

- **Computed on every push to `main`.** A workflow in the monorepo lists the folders under the four top-level folders that hold an `OWNERS` file and creates each missing label. It needs no checked-in file, so a new project folder gets its label with the commit that adds it.
- **One identical set in both repositories.** The monorepo's tree is the only source. The companion's workflow reads the same list from the monorepo commit it pins and creates the same labels, with `area/repo`, `area/third_party`, and `area/none` among them. The companion's own tree adds none, so no private name becomes a label in either repository.
- **The nearest folder with an `OWNERS` file decides.** A path takes the label of the nearest folder at or above it that holds an `OWNERS` file in the monorepo, and a companion path is read at its mirrored public path. An issue with no such folder is `area/repo` when it concerns the root and `area/none` otherwise. A companion test fails for an `OWNERS` file at a path the monorepo doesn't have.
- **Bounded.** GitHub caps a label's name at 50 characters (per docs, unverified here), so a test fails for a project folder whose label would be longer.
- **Kept while in use.** When a project folder goes away, keep its retired label in both repositories until no open issue in either carries it. The sync check fails for each of those issues until it's relabeled.
- **Separate from review and from CI.** An area label doesn't set a review tier, names no approver, and doesn't pick the record lane. The tier 2 path map does the first, the reviews App and the tier 0 workflow approve by tier, and a path check on the diff does the third ([appendix C](C-layout-and-ownership.md#separation-of-duties)).

Two things enforce the one-label rule. The issue commands require an area on `create` and swap it in one step on `edit`, and they refuse a label the tree doesn't produce. A sync check then runs on each issue event and once a day in both repositories. It fails for any open issue with no area label, with several, or with one whose folder is gone. That check is a new rule for the [issue checks design](../issue-checks/README.md), which already reads `area/` labels.

## Issue rules

Five rules come with the move (D32). The issue commands in `factory/tbhbdev` enforce each one (new), and the [tracker design](../pm/README.md) holds their detail.

- **Status changes only through the issue commands.** A state machine defined in code lists the allowed moves, and the commands refuse any other. The sync check reports a Status that changed any other way.
- **One agent run for each issue at a time.** A run takes the issue before it starts, and the issue shows a visible mark while the run holds it. Parallel work on one piece of work comes from splitting it into Task issues, never from two runs on one issue.
- **One hierarchy.** An Initiative holds Epics, an Epic holds Features, and a Feature may hold Tasks. The Task level is optional. Defects, Chores, Research, and Spikes may have no parent.
- **Several pull requests may fulfil one issue.** A pull request links its issue with a `Refs:` line, or with `Completes:` when it's the last one.
- **Only a guarded `issue close` closes an issue.** The pull request template and the body check reject GitHub's closing keywords, so no merge closes an issue by itself. The guard's conditions belong to the tracker design, and this design doesn't set them.

## The relevance pass

No existing issue moves as it is. Two sets get a relevance pass first (D28):

- **The tracker files.** The operator's file tracker holds issue files. This design states no count of them and none of the issues that stay, because the relevance pass reads the files and produces both.
- **Open GitHub issues in the repositories that move in.** Their count wasn't read, because this pass had no access to the GitHub API (unverified).

`tbhb-agent` sorts both sets using the same criteria (D25). A fresh `tbhb-agent` run on another provider, with no context from the sorting run, confirms each close list before any issue on it closes (D34). Each close carries a one-line reason, and each kept issue carries one area label.

| Outcome | When |
| --- | --- |
| Close | The work is done, duplicated, or abandoned, or it concerns something the move removes: a pin update, a caller workflow, duplicated configuration, or a sync between repositories |
| Keep, public | The work is still wanted, and the issue can be stated with no private detail |
| Keep, private | The work is still wanted, and stating it needs a host, an account, an inventory value, or another private detail |

The pass decides public or private for each issue, not for each repository. A private repository's issue can be public, and a public tracker file can turn out to need a private detail.

### From a source repository to a label

A source repository doesn't always map to one label, so the pass also picks each kept issue's area:

| Case | Area label |
| --- | --- |
| A repository that becomes one project | That project's label |
| A private twin of a public repository, such as the private wiki | The public project's label, in the companion |
| A repository split between the monorepo and the companion, such as ops | Chosen for each issue, and the label is the same project on either side |
| A work item about one project | That project's label |
| A work item about the repository's root, or about several projects | `area/repo` |
| An issue that belongs to no project | `area/none` |

## Importing the survivors

| Source | Destination | How |
| --- | --- | --- |
| A kept tracker file | Either repository | `tbhbdev alpha issue create`, through the staging steps in the [tracker import design](../pm/E-import.md) |
| An open issue in a public repository, kept public | The monorepo | `gh issue transfer`, which keeps its comments, then the area label |
| An open issue in a public repository, kept private | The companion | A new issue in the companion that links the old one and holds the private detail. The old issue closes with a reason that names no private detail |
| An open issue in a private repository, kept private | The companion | `gh issue transfer`, then the area label |
| An open issue in a private repository, kept public | The monorepo | A new issue with the private details removed. GitHub doesn't transfer an issue from a private repository to a public one (per docs) |

The kept tracker issues import in plan step 5, and the import takes whatever the final relevance pass keeps. That's after this design merges and after the scaffolding has created the labels, and it doesn't wait for any code import. The scaffolding creates the folder, the `README.md`, and the `OWNERS` file of every planned project, so every area label exists before its code arrives.

The import adds every survivor to the main private Project and every monorepo survivor to the public roadmap. It paces its writes under GitHub's limits on content creation, and it records the new issue's number against each tracker file. A second run skips a file that already has a number.

## The archive gate

No old repository is archived while it has an open issue or an open pull request (D22). `tbhbdev alpha monorepo archive <repository>` refuses unless the import commit that names the repository is on the monorepo's `main`, `gh issue list --state open` and `gh pr list --state open` both return nothing, and the repository's README points to the new path.
