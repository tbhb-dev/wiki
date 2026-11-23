---
type: Design Appendix
title: Renovate auto-approval decisions
description: The open and decided questions of the Renovate auto-approval design.
tags: [renovate, approvals, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Renovate auto-approval decisions

Back to [the design](README.md).

## Open

### D2. Does the workflow's approval replace the one review run that a first-party pin gets today?

- **Recommendation:** yes, for a first-party pin when the workflow proves its source's default branch contains the new commit.
- **Why:** [Architecture Decision Record (ADR) 0001](../../adrs/0001-the-changed-paths-and-content-set-the-review-tier.md) and [ADR 0024](../../adrs/0024-renovate-pull-requests-automerge-after-approval-and-ci.md) give a pin to a reviewed and merged commit one review run. The workflow checks that condition mechanically, and the daily `pins` group carries the tbhb-dev/rules pins, so without this the group rarely qualifies.
- **If yes:** plan step 1 writes a decision record that amends both records for pin-only Renovate pull requests, and the matcher accepts first-party branch pins that pass the merged-at-source check.
- **If no:** the matcher refuses every first-party pin with `first_party_needs_review`. Third-party pins still qualify, and any pull request that moves a tbhb-dev pin keeps its review run.
- **Reversible:** yes. A later decision record restores the review run, and one input default turns first-party pins off.

### D3. May the administrator create the approver App and its environments?

- **Recommendation:** yes, as `tbhb-dev-pin-approver`, with pull requests write, contents read, checks read, and GitHub's mandatory metadata read, installed on selected repositories only, with no webhook.
- **Why.** The workflow needs its own identity to approve pull requests independently of Renovate. The environment limits access to the credential to jobs that use the default branch ref.
- **If yes.** In each calling repository, create `pins-approver` and allow deployments only from the default branch. Leave required reviewers unset. Add `PIN_APPROVER_CLIENT_ID` and `PIN_APPROVER_PRIVATE_KEY` as environment secrets. GitHub counts approvals only from reviewers with write permission. If the spike shows that the App needs contents write for its approval to count, the operator decides whether to grant it. [Setup steps](D-workflow.md#what-the-administrator-sets-up)
- **If no:** nothing gets an automatic approval, and pin-only pull requests keep waiting for a hand approval.
- **Reversible.** Revoke the App's private key and uninstall the App to stop approvals across repositories. Remove both environment secrets from `pins-approver` in each calling repository. Past reviews stay on their pull requests.

### D4. Should the matcher also accept the renovate-config preset pins in `.github/renovate.json5`?

- **Recommendation:** yes, after the first repository's evaluate period, under the same merged-at-source rule as a first-party `uses:` pin.
- **Why:** the `pins` group moves the preset pins and the renovate-config workflow pins together. The workflow pin is in scope and the preset pin names the same commit, so refusing one and accepting the other refuses the whole group anyway.
- **If yes:** the matcher gains a `renovate-preset` line kind, with its own accept and refuse cases, in the release after the evaluate period.
- **If no:** any `pins` pull request that touches `.github/renovate.json5` gets `not_a_pin_file` and takes the normal path.
- **Reversible:** yes, by removing the line kind in a later release.

## Decided

### D12. Where do the Renovate App's identity values come from?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule:** the Renovate App's ID, its bot user ID, and its bot login are constants in the decision module, which the rules maintainer sets from public GitHub API reads when building the module. No input or variable can change them.
- **Undo:** two-way door. A later release can move them to inputs.
- **Record:** this design's pull request.

### D11. Does the workflow approve a pull request that moves its own reusable workflow pin?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule:** yes, under the same rules as any first-party pin. The run that decides is always the default branch's caller, so a new version of the workflow never judges its own adoption.
- **Undo:** two-way door. A `self_pin` refusal is one rule in the matcher.
- **Record:** this design's pull request.

### D10. Where do the code and tests live?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule:** in tbhb-dev/rules, as a standard-library-only Python package at `repo/private/renovate_approval/`, with `py_test` targets beside it and fixtures in `testdata/`.
- **Undo:** two-way door. Moving the package changes one path in the reusable workflow.
- **Record:** this design's pull request.

### D9. How does the matcher treat lockfiles?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule:** `mise.lock` and `MODULE.bazel.lock` are companions. The matcher accepts a change to one only when the same pull request moves the pin it belongs to, compares the parsed base and head documents, and refuses any change outside the entries [appendix C](C-matcher.md#lockfiles) lists. The matcher refuses every other lockfile.
- **Undo:** two-way door, by widening the entry list in a later release.
- **Record:** this design's pull request.

### D8. Does a version bump with a matching digest count as pin-only?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule:** yes, when the version and the digest change in the same stanza, the URL changes only by that version, and the new digest equals the SHA-256 hash, from the Secure Hash Algorithm (SHA) family, of the artifact the run downloads. A version change without a digest change, or a digest change without a version change, gets a refusal.
- **Undo:** two-way door. Refusing all archive pins is one rule.
- **Record:** this design's pull request.

### D7. Which bump levels qualify by default?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule:** `patch`, `minor`, and `digest` qualify, and `major` doesn't. A repository may narrow the list through `allowed_bumps`, and widening it to `major` takes a change to that repository's caller, which its review tier covers.
- **Undo:** two-way door, by changing the default.
- **Record:** this design's pull request.

### D6. Where does the diff come from?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule:** the run fetches the merge base and H without blobs or a checkout, then runs `git diff --unified=0` with rename detection and external tools off. It cross-checks the file list against the pull request files API.
- **Undo:** two-way door. The API's patch fields are the fallback, and they lose the rename and mode detail.
- **Record:** this design's pull request.

### D5. Which event triggers the workflow?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule:** `workflow_run` with `types: [completed]`, listing each workflow that reports a required check. `pull_request_target` and `check_suite` aren't used, for the reasons in [appendix D](D-workflow.md#the-trigger).
- **Undo:** two-way door. The caller's `on:` block is the only change.
- **Record:** this design's pull request.

### D1. How do pin-only Renovate pull requests get their approval?

- **Decided:** 2026-11-23, by the operator.
- **Rule:** pin-only Renovate pull requests keep their approval requirement. A reusable workflow, which each repository calls, gives that approval only when the author is the Renovate App, every changed hunk only replaces a pinned version or digest in a known pin file, the required checks succeeded at the head, and the head hasn't moved. A dedicated GitHub App posts the approval, never Renovate's identity, and the workflow fails closed and logs its reason.
- **Undo:** two-way door. Disabling the callers and uninstalling the App returns every Renovate pull request to hand approval.
- **Record:** the operator's direction to the design's author on 2026-11-23. This design's pull request is its first written record, and plan step 1 turns it into a decision record.
