<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Decision records

This public repository holds the decision records that can be stated without anything private. An ADR goes here whenever it can be sanitized without losing its meaning: host names, the tailnet, IP addresses, internal paths, account names and credentials are removed or made generic, and the rule still reads the same. Only a record that would lose its meaning goes in the private wiki repository's `adrs/` folder.

A decision record (ADR) states one decision as a rule in the present tense, with the context that led to it and what follows from it. A record makes sense to a reader of this repository alone.

## Layout

Each record is `adrs/NNNN-<slug>.md`: the next free number, zero-padded to four digits, then a kebab-case slug of the title, such as `0007-squash-merge-every-pull-request.md`. This repository numbers its own records from `0001`. [TEMPLATE.md](TEMPLATE.md) is the house template, and the format and writing rules are in the decision-records design.

## Rules

- A record never names a host, the tailnet, a vault, an internal path, an account, a private repository or a public IP address. It uses role labels such as "the Linux build host".
- A record never cites a Decisions note or links to a private repository.
- A record never quotes anyone or says who decided. Acceptance is the pull request's approval and merge, and the pull request cites the source of the decision.
- The `decided` field is the date of the decision, not of the record.
- Nobody renames a record after it merges, with one exception: the fix for a duplicate number renames the later file.

## Records

Add each new record's entry at the end of this list, as `[title](NNNN-slug.md)` and the record's description. When two pull requests add a record at once, the later one rebases and takes the next number.

- [Issue priority is P0 to P3, set by agents with provenance](0001-issue-priority-is-p0-to-p3-set-by-agents-with-provenance.md): Every issue's priority is one of P0 to P3 in the org Priority field, and agents set it and record why.
- [Work state lives in GitHub Issues and Projects](0002-work-state-lives-in-github-issues-and-projects.md): Work items are GitHub issues gathered in org Projects, agents open them, and an issue may open whenever the work is known.
- [A work item is an issue in the repository that owns the work](0003-a-work-item-is-an-issue-in-the-repository-that-owns-the-work.md): Bugs and tasks live in the repository that owns the work, Initiatives, Epics and Stories live in the central work repository, and org Projects gather them.
- [An issue goes on the public twin by default](0004-an-issue-goes-on-the-public-twin-by-default.md): For a repository with a public and a private twin, an issue goes on the public one unless its text needs private details, and a scan decides before it's created.
- [Asks to the operator or between roles are Request issues](0005-asks-to-the-operator-or-between-roles-are-request-issues.md): A request for a decision, an action, access or information is a Request issue, whether the operator or an agent handles it, and session messages stay for steering and urgent stops.
- [Service-management work runs as issues, and final reports land as record files](0006-service-management-work-runs-as-issues-and-final-reports-land-as-record-files.md): Incidents, Problems, Requests and Changes run their workflows as issues, while final reports and configuration items land as record files by pull request with no shared index in the diff.
- [The org issue taxonomy adds Problem, Request and Change](0007-the-org-issue-taxonomy-adds-problem-request-and-change.md): The org has ten issue types, the original seven plus Problem, Request and Change, with one Severity definition and the shared fields and label families.
- [Changes follow three ITIL classes, and approval follows the constitution](0008-changes-follow-three-itil-classes-and-approval-follows-the-constitution.md): A Change is a Standard, Normal or Emergency change with a plan, risk, rollback and verification, and the owning role approves it unless it crosses roles or touches a reserved matter.
- [Content writes go only through managed commands and the content kernel](0009-content-writes-go-only-through-managed-commands-and-the-content-kernel.md): Every issue, comment, Project and pull request text write goes through a managed command built on the shared content kernel, and hooks block bare GitHub writes.
- [Issue comments are append-only work notes](0010-issue-comments-are-append-only-work-notes.md): A comment on a service-management issue is a typed, append-only work note written from a template, and every state transition writes its own note.
- [Templates give starting text, and CUE schemas decide validity](0011-templates-give-starting-text-and-cue-schemas-decide-validity.md): Each content type has a template file for its starting text and a CUE schema that decides whether text is valid, with a short named list of kernel checks for rules CUE can't express.
- [pm owns decision records end to end](0012-pm-owns-decision-records-end-to-end.md): The pm repository owns the decision-record type, the adr command and the adrs/ rules, and service management keeps only the decision Request that can lead to a record.
- [pm owns the templates and checks for review-run output](0013-pm-owns-the-templates-and-checks-for-review-run-output.md): What a review run posts on a pull request follows templates and mechanical checks that pm owns in the content kernel.
- [The service-management and project-management designs live in the public wiki](0014-the-service-management-and-project-management-designs-live-in-the-public-wiki.md): Designs go in the wiki repositories, not product repositories, and the service-management and project-management designs go in the public wiki under the public-repository rules.
