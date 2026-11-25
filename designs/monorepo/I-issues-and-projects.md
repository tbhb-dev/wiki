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
| The roadmap Project | Public | Only issues from the monorepo (decided) |
| The private Project | Private | Issues from both repositories (proposed, D26) |

`tbhbdev alpha issue create` adds each new issue to its Projects, and the sync check below reports any that a person filed by hand. The scaffolding creates both Projects in plan step 4.

## Area labels

Each issue carries exactly one area label (D28). The label names a project, and the generator computes the projects from the tree. No list file exists, and no label comes from an `OWNERS` file.

| Label | One for each |
| --- | --- |
| `area/factory/<project>` | direct child folder of `factory/` |
| `area/products/<name>` | direct child folder of `products/` |
| `area/lib/<name>` | direct child folder of `lib/` |
| `area/docs/<name>` | direct child folder of `docs/` |
| `area/repo` | the repository, once: CI, `.github/`, rulesets, `MODULE.bazel`, and the language roots |
| `area/third_party` | the repository, once: everything under `third_party/` |
| `area/none` | the repository, once: an issue that belongs to no project |

An issue about the event gateway's retry logic is `area/factory/event-gateway`. An issue about the command-line tool is `area/factory/tbhbdev`.

A project's label covers the project's own records. An issue about a need, a requirement, or a knowledge article under `factory/tbhbdev/docs` is `area/factory/tbhbdev`, not `area/docs/wiki` (D33). The `area/docs/<name>` labels cover only the wiki, research, and spikes, and a project's `docs/` folder never gets a label of its own.

- **Computed on every push to `main`.** A workflow lists the direct children of the four folders and creates each missing label. It needs no checked-in file, so a new project folder gets its label with the commit that adds it.
- **One set for each repository, from its own tree.** The monorepo's labels come from the public tree only. The companion's set is the public set, read from the monorepo commit it pins, plus one label for each project folder that exists only in the companion. No private project's name becomes a public label.
- **Bounded.** GitHub caps a label's name at 50 characters (per docs, unverified here), so a test fails for a project folder whose label would be longer.
- **Kept while in use.** When a project folder goes away, its label stays until no open issue carries it. The sync check fails for each of those issues until it's relabeled.
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

- **The tracker files.** The operator's file tracker held 804 issue files. The pass over them kept 742 (the brief says; this pass didn't count them and can't derive the figure).
- **Open GitHub issues in the repositories that move in.** Their count wasn't read, because this pass had no access to the GitHub API (unverified).

D25 asks the operator to confirm the criteria and the check on each close list. The proposed criteria sort each issue into one of three outcomes, with a one-line reason on each close and one area label on each kept issue:

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
| An open issue in a public repository | The monorepo | `gh issue transfer`, which keeps its comments, then the area label |
| An open issue in a private repository, kept private | The companion | `gh issue transfer`, then the area label |
| An open issue in a private repository, kept public | The monorepo | A new issue with the private details removed. GitHub doesn't transfer an issue from a private repository to a public one (per docs) |

The 742 kept tracker issues import in plan step 5. That's after this design merges and after the scaffolding has created the labels, and it doesn't wait for any code import. The scaffolding creates the folder, the `README.md`, and the `OWNERS` file of every planned project, so every area label exists before its code arrives.

The import paces its writes under GitHub's limits on content creation, and it records the new issue's number against each tracker file. A second run skips a file that already has a number.

## The archive gate

No old repository is archived while it has an open issue or an open pull request (D22). `tbhbdev alpha monorepo archive <repository>` refuses unless the import commit that names the repository is on the monorepo's `main`, `gh issue list --state open` and `gh pr list --state open` both return nothing, and the repository's README points to the new path.
