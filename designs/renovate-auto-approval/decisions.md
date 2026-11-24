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

None.

## Decided

### D12. Where do the Renovate App's identity values come from?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule:** the Renovate App's ID, its bot user ID, and its bot login are constants in the decision module, which the rules maintainer sets from public GitHub API reads when building the module. No input or variable can change them.
- **Undo:** two-way door. A later release can move them to inputs.
- **Record:** this design's pull request.

### D11. Does the workflow approve a pull request that moves its own reusable workflow pin?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule.** Yes, when the pin appears in a separate caller and passes the [first-party rules in appendix C](C-matcher.md#workflow-uses-lines), subject to D2. Changes in the file that defines the reusable approver refuse with `self_pin`. The caller on the default branch selects the reusable workflow, and the job uses `job.workflow_repository` and `job.workflow_sha` to verify that exact revision before requesting the token.
- **Undo.** Two-way door. Widen the `self_pin` refusal to cover references to the reusable approver from callers.
- **Record:** this design's pull request.

### D10. Where do the code and tests live?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule:** in tbhb-dev/rules, as a standard-library-only Python package at `repo/private/renovate_approval/`, with `py_test` targets beside it and fixtures in `testdata/`.
- **Undo:** two-way door. Moving the package changes one path in the reusable workflow.
- **Record:** this design's pull request.

### D9. How does the matcher treat lockfiles?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule.** `mise.lock` and `MODULE.bazel.lock` are companions. The matcher accepts only the parsed entry changes [appendix C](C-matcher.md#lockfiles) permits for a pin the same pull request moves. That includes standalone Bazel registry additions, and a mise entry's `url_api` must identify the same artifact as its download URL.
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
- **Rule:** the run fetches the merge base and H without blobs or a checkout, then runs `git diff --unified=0` with rename detection and external tools off. It requires both file modes from the raw diff and cross-checks the files against the list GitHub reports for the pull request.
- **Undo:** two-way door. A replacement source needs to supply the same rename and mode evidence.
- **Record:** this design's pull request.

### D5. Which event triggers the workflow?

- **Decided:** 2026-11-23, by the rules maintainer.
- **Rule.** The caller uses `workflow_run` with `types: [completed]` and lists one CI workflow, which must report all required checks. The matcher returns `multiple_required_workflows` if required checks come from more than one workflow. [Trigger and required checks](D-workflow.md#the-trigger)
- **Undo.** Two-way door. A later release can change the trigger or extend workflow support.
- **Record:** this design's pull request.

### D4. Should the matcher also accept the renovate-config preset pins in `.github/renovate.json5`?

- **Decided.** The operator made this choice on 2026-11-23.
- **Rule.** Keep the pilot's existing pin-file list. A pull request that touches `.github/renovate.json5` gets `not_a_pin_file` and follows the normal review path. Adding the preset pins widens what the workflow can approve, so the rules maintainer takes the evidence to the operator after the first repository's evaluate period. [Rollout follow-up](F-rollout.md#follow-up)
- **Undo.** The operator can authorize a separate change to add the preset pins after reviewing the pilot evidence.
- **Record.** See the operator's decision of 2026-11-23.

### D3. May the administrator create the approver App and its environments?

- **Decided.** The operator made this choice on 2026-11-23.
- **Rule.** The administrator creates `tbhb-dev-renovate-approver` with the least permissions that make its review count. Start with pull requests write, contents read, checks read, and GitHub's mandatory metadata read. Install it only on selected repositories and leave its webhook off.
- **Environment.** In each calling repository, create `pins-approver` with a run-ref restriction to the default branch and leave required reviewers unset. Store both App secrets in that environment. The spike tests the restriction and whether the review counts, and any request for a wider grant goes to the operator. [Setup steps](D-workflow.md#what-the-administrator-sets-up) and [spike checks](A-evidence.md#open-questions-for-the-spike).
- **Undo.** Revoke the App's private key, uninstall the App, and remove both secrets from each `pins-approver` environment. Past reviews stay on their pull requests.
- **Record.** See the operator's decision of 2026-11-23.

### D2. Does the workflow's approval replace the one review run that a first-party pin gets today?

- **Decided.** The operator made this choice on 2026-11-23.
- **Rule.** A qualifying pin-only Renovate pull request gets its approval from the reusable workflow's App. That approval replaces the one review run for a first-party pin, and the workflow must prove that the source's default branch contains the new commit. Pull requests that fail any qualification follow the normal review path. [First-party pin rules](C-matcher.md#workflow-uses-lines)
- **Undo.** Restore the first-party review run in a later decision record and stop automatic approval for first-party pins.
- **Record.** See the operator's decision of 2026-11-23.

### D1. How do pin-only Renovate pull requests get their approval?

- **Decided:** 2026-11-23, by the operator.
- **Rule.** Pin-only Renovate pull requests still require approval. A reusable workflow gives that approval when the Renovate App authored the pull request, the diff contains only recognized pin moves and their permitted companion lockfile changes, and the required checks passed in that pull request's CI snapshot. The workflow confirms the head and base again before a dedicated GitHub App posts the review. It fails closed and logs its reason.
- **Undo:** two-way door. Disabling the callers and uninstalling the App returns every Renovate pull request to hand approval.
- **Record:** the operator's decision of 2026-11-23. Plan step 1 records the rule in an Architecture Decision Record (ADR).
