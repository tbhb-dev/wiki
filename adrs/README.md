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

- [Review depth follows tiers set by the paths a diff touches](0001-review-depth-follows-tiers-set-by-the-paths-a-diff-touches.md): A pull request gets no review run, one, or two, by a tier that the paths and content of its diff set, and nothing lowers a tier.
- [Tier 0 pull requests merge on green without review or approval](0002-tier-0-pull-requests-merge-on-green-without-review-or-approval.md): The session that commissioned a tier 0 pull request merges it once its head is green, with no review, approval or named merger.
- [Reviews come from a provider other than the builder's](0003-reviews-come-from-a-provider-other-than-the-builders.md): Each review run uses a model from a provider other than the one that built the commits it covers, with a same-provider fallback one tier up.
- [The session that commissions a build may launch its reviews but never judges them](0004-the-session-that-commissions-a-build-may-launch-its-reviews-but-never-judges-them.md): The maker is the build run, so the session that commissioned it may launch fresh review runs, but it never writes, reclassifies or edits a verdict.
- [Review runs post their own verdicts on the pull request](0005-review-runs-post-their-own-verdicts-on-the-pull-request.md): Each review run posts its verdict on the pull request for an exact head, and whoever approves or merges reads it there, never from a relay.
- [Reviews launch on check events, with no per-repository queue](0006-reviews-launch-on-check-events-with-no-per-repository-queue.md): A green conclusion check on a held head launches its reviews at once, every open pull request is reviewed in parallel, and only host capacity limits it.
- [A finding blocks only when normal use breaks](0007-a-finding-blocks-only-when-normal-use-breaks.md): A review finding is a BLOCK only when ordinary use with no adversary gives a wrong, destructive or broken result, and other findings become logged debt.
- [Code written to dodge a check is a blocking finding](0008-code-written-to-dodge-a-check-is-a-blocking-finding.md): Code shaped so a scanner, lint or check won't match is always a blocking review finding, fixed at the cause or through a reviewed allowlist entry.
- [The reviews App approves, never for the session that opened the pull request](0009-the-reviews-app-approves-never-for-the-session-that-opened-the-pull-request.md): Approvals come from the reviews GitHub App after a host-side check that the requester didn't open the pull request, and never with an unverified commit.
- [A rebuilt pull request whose tree matches an approved one is approved without a model read](0010-a-rebuilt-pull-request-whose-tree-matches-an-approved-one-is-approved-without-a-model-read.md): The review and merge worker approves a replacement pull request with no new review run when its tree equals a tree an earlier review approved.
- [Review reads each repository's review files from the base branch](0011-review-reads-each-repositorys-review-files-from-the-base-branch.md): The review tooling reads a repository's review config, PR template, checklist and schemas from the pull request's base branch, never its head.
- [Agents dismiss reviews and close pull requests, with provenance](0012-agents-dismiss-reviews-and-close-pull-requests-with-provenance.md): Agents have full control over dismissing reviews and closing pull requests as long as each act records its provenance, and merges carry a Merged-by trailer.
- [Repeated change requests trigger a design assessment, not a human handoff](0013-repeated-change-requests-trigger-a-design-assessment-not-a-human-handoff.md): Three rounds of requested changes on a pull request don't hand it to a human automatically, but they do trigger an assessment of whether the design or requirements are flawed.
- [The review and merge worker is a Unix service account and adds no GitHub user](0014-the-review-and-merge-worker-is-a-unix-service-account-and-adds-no-github-user.md): The review and merge worker runs as a Unix service account with no agent harness and no sudo, and writes to GitHub only through the existing Apps.
- [The review and merge worker may read GitHub every 15 minutes](0015-the-review-and-merge-worker-may-read-github-every-15-minutes.md): The review and merge worker reads GitHub every 15 minutes to catch lost events, as a sanctioned exception to the rule against polling.
