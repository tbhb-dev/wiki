---
type: Design Appendix
title: One public monorepo for the factory issues and Projects
description: Where issues live after the move, the two Projects, the generated area labels, the relevance pass over existing issues, and the gate before an old repository is archived.
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

`tbhbdev alpha issue create` adds each new issue to its Projects, and the sync check below repairs any that a person filed by hand. The scaffolding creates both Projects in plan step 4.

## Area labels

Each issue carries exactly one `area/<path>` label, where the path is the nearest folder that holds an `OWNERS` file (D28). An issue about the event gateway's retry logic is `area/factory/services/event-gateway`. A file at the repository root has the root `OWNERS` file as its nearest, and that label is `area/root`.

- **Generated.** The generator that writes `CODEOWNERS` also writes `factory/build/policy/labels.yaml`, with one label for each `OWNERS` file. A test fails when the file is stale.
- **Identical in both repositories.** A workflow on each push to `main` applies the file to the monorepo's labels. The companion's workflow applies the same file from its pinned monorepo commit. The companion has no `OWNERS` files of its own, so it can't add a label.
- **As fine as the owners.** An area is as fine as the `OWNERS` layout. Every project root is an area, including each `products/<name>` and each `lib/<name>`, and a project gets a finer area by adding an `OWNERS` file to a subfolder ([appendix C](C-layout-and-ownership.md#owners-files)).
- **Routed.** The label names the folder, and the folder's `OWNERS` file names the role, so an issue's owner follows from its label.
- **Bounded.** GitHub caps a label's name at 50 characters (per docs, unverified here), so the generator's test fails for a longer path.
- **Kept while in use.** When an `OWNERS` file moves or goes away, its label stays until no open issue carries it, and the sync check lists those issues for the owning role to relabel.

Two things enforce the one-label rule. The issue commands require an area on `create` and swap it in one step on `edit`, and they refuse a label that isn't in the generated file. A sync check then runs on each issue event and once a day in both repositories, and it reports any open issue with no area label, more than one, or one whose folder is gone. That check is a new rule for the [issue checks design](../issue-checks/README.md), which already reads `area/` labels.

## The relevance pass

No existing issue moves as it is. Two sets get a relevance pass first (D28):

- **The tracker files.** The operator's file tracker held 804 issue files. The pass over them kept 742 (the brief says; this pass didn't count them).
- **Open GitHub issues in the repositories that move in.** Their count wasn't read, because this pass had no access to the GitHub API (unverified).

D25 asks who runs the pass and confirms the criteria. The proposed criteria sort each issue into one of three outcomes, with a one-line reason on each close and an area on each kept issue:

| Outcome | When |
| --- | --- |
| Close | The work is done, duplicated, or abandoned, or it concerns something the move removes: a pin update, a caller workflow, duplicated configuration, or a sync between repositories |
| Keep, public | The work is still wanted, and the issue can be stated with no private detail |
| Keep, private | The work is still wanted, and stating it needs a host, an account, an inventory value, or another private detail |

## Importing the survivors

| Source | Destination | How |
| --- | --- | --- |
| A kept tracker file | Either repository | The imports role creates it with `tbhbdev alpha issue create`, through the staging steps in the [tracker import design](../pm/E-import.md) |
| An open issue in a public repository | The monorepo | `gh issue transfer`, which keeps its comments, then the area label |
| An open issue in a private repository, kept private | The companion | `gh issue transfer`, then the area label |
| An open issue in a private repository, kept public | The monorepo | A new issue with the private details removed. GitHub doesn't transfer an issue from a private repository to a public one (per docs) |

The 742 kept tracker issues import in plan step 5. That's after this design merges and after the scaffolding has generated the labels, and it doesn't wait for any code import. The scaffolding creates the `OWNERS` file for every planned path, so every area exists before its code arrives.

The import paces its writes under GitHub's limits on content creation, and it records the new issue's number against each tracker file. A second run skips a file that already has a number.

## The archive gate

No old repository is archived while it has an open issue or an open pull request (D22). `tbhbdev alpha monorepo archive <repository>` refuses unless the import commit that names the repository is on the monorepo's `main`, `gh issue list --state open` and `gh pr list --state open` both return nothing, and the repository's README points to the new path.
