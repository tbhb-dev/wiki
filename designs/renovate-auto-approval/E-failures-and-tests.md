---
type: Design Appendix
title: Renovate auto-approval failure modes and tests
description: Every failure mode, its reason code, and the table-driven test plan for the decision function.
tags: [renovate, approvals, testing, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Renovate auto-approval failure modes and tests

Back to [the design](README.md).

## The fail-closed rule

Every path out of the run ends in one of two outcomes: an approval with reason `ok_pin_only`, or no approval with exactly one other reason code. An unexpected exception anywhere maps to `internal_error`. The entry point catches it, writes the summary, and exits with status 0, so a refusal never reads as a broken workflow. A refusal never retries inside the run. The next CI completion, or a re-run, decides again from scratch.

The job summary shows the mode, the pull request number, H, the decision, the reason code, every finding with its file and line, and the time of each of the three head reads. `decision.json` carries the same fields for the rollout review in [appendix F](F-rollout.md).

## Failure modes

| Failure | What the run does | Reason code |
| --- | --- | --- |
| A GitHub API call returns 5xx or times out | Retries twice with backoff, then refuses | `api_error` |
| A call returns 403 or 429 with no remaining quota or a `retry-after` | Refuses at once without waiting | `rate_limited` |
| A call returns 404 for the pull request or a commit | Refuses | `api_error` |
| A paginated list ends early or its count disagrees with the total | Refuses | `pagination_truncated` |
| The push that triggered CI isn't the pull request's head when the run starts | Refuses, and the newer head's CI run decides | `head_mismatch_at_start` |
| A force-push lands during the gather step | The second read sees it and refuses | `head_moved` |
| A force-push lands between the second read and the post | The ruleset dismisses the approval, and the third read dismisses it too | `head_moved_after_approval` |
| The pull request closes or merges by hand during the run | The second read sees it and refuses | `pr_not_open` |
| A pull request changes 500 files | Refuses from the files count, before any fetch | `too_many_files` |
| The API truncates a large diff | The run doesn't use the API's diff. It computes the diff with Git, and cross-checks the file list | `file_list_mismatch` |
| The Git fetch of H or the merge base fails | Refuses | `diff_unavailable` |
| The computed diff exceeds 1 MB | Refuses before parsing | `diff_too_large` |
| An artifact download fails, or runs over `max_download_mb` | Refuses | `digest_unverifiable` |
| A tag or ancestry lookup fails | Refuses | `fact_missing` |
| A Renovate group pull request, such as the daily `pins` group | Decides on the whole diff. One refused hunk refuses the whole pull request, and the summary lists every finding | the first finding's code |
| Auto-merge merges within seconds of the approval | The third read accepts a merged pull request whose head is still H | `ok_pin_only` |
| The approval post fails | Refuses, and nothing needs undoing | `approval_failed` |
| The dismissal after a moved head fails | Logs both codes, and the ruleset's stale dismissal still applies | `head_moved_after_approval`, `dismissal_failed` |
| The App already approved H | Posts nothing | `already_approved` |
| A pull request carries the opt-out label | Refuses before gathering | `opted_out` |
| The ruleset doesn't dismiss stale approvals or require last-push approval | Refuses | `stale_dismissal_off` |
| The job hits its 10-minute timeout | GitHub stops the job, no approval exists, and the summary may be empty | none, the run shows as timed out |
| Unexpected exception | Refuses | `internal_error` |

## Reason codes

The codes group by the stage that sets them. `decide()` sets the pull request, diff, pin, and check codes. The gather step and the approval step set the rest, before or after `decide()`.

| Stage | Codes |
| --- | --- |
| Approve | `ok_pin_only` |
| No-op | `already_approved` |
| Pull request | `pr_not_unique`, `pr_not_open`, `pr_draft`, `base_not_default`, `fork_head`, `opted_out`, `author_not_renovate`, `author_app_mismatch`, `commit_not_renovate`, `signature_unverified`, `too_many_commits` |
| Diff, file level | `empty_diff`, `too_many_files`, `diff_too_large`, `file_list_mismatch`, `not_a_pin_file`, `rename`, `copy`, `new_file`, `deleted_file`, `mode_change`, `binary`, `submodule_or_symlink`, `eol_change`, `encoding` |
| Diff, hunk level | `line_added`, `line_removed`, `line_count_changed`, `not_a_pin_line`, `mixed_line_kinds`, `hunk_spans_kinds`, `key_changed`, `no_value_change`, `parse_doubt` |
| Pin values | `comment_only_change`, `unversioned_pin`, `unpinned_version`, `unparsed_version`, `downgrade`, `bump_level_not_allowed`, `digest_without_version_change`, `digest_not_updated`, `digest_mismatch`, `url_changed_beyond_version`, `tag_mismatch`, `not_first_party`, `not_merged_at_source`, `first_party_needs_review` |
| Lockfiles | `lock_not_updated`, `lock_changed_without_pin`, `lock_version_mismatch`, `lock_key_changed` |
| Checks and rules | `checks_pending`, `check_skipped`, `check_neutral`, `check_failed`, `check_missing`, `required_check_unbound`, `no_required_checks`, `self_check_required`, `stale_dismissal_off` |
| Head binding | `head_mismatch_at_start`, `head_moved`, `head_moved_after_approval` |
| Gather and API | `api_error`, `rate_limited`, `pagination_truncated`, `diff_unavailable`, `digest_unverifiable`, `fact_missing`, `approval_failed`, `dismissal_failed`, `internal_error` |

`first_party_needs_review` exists only if the operator answers no to [D2](decisions.md).

## Where the code lives

The package sits at `repo/private/renovate_approval/` in tbhb-dev/rules, beside the other repository checks, as recorded in [D10](decisions.md). It imports only the Python standard library, so the runner installs nothing.

| File | Contents |
| --- | --- |
| `decide.py` | `decide(diff, metadata, checks) -> Decision`. Pure: no network, no clock, no environment |
| `matcher.py` | The diff parser and one grammar per pin kind, called by `decide.py` |
| `versions.py` | Version parsing and bump levels |
| `gather.py` | The API and Git calls that build the three inputs |
| `approve.py` | The second read, the post, and the third read |
| `__main__.py` | The entry point that wires them and writes the summary and `decision.json` |
| `test_decide.py`, `test_matcher.py`, `test_versions.py` | Table-driven tests |
| `testdata/` | One directory per case: `diff.patch`, `metadata.json`, `checks.json`, and any base and head file text |

`BUILD.bazel` declares `py_test` targets for the three test files. They run in `bazelisk test //...` with no network.

`Decision` holds `approve`, `reason`, and `findings`, a list of `(code, path, line, detail)`. `approve` is true only when `reason` is `ok_pin_only` and `findings` is empty. A test asserts that invariant over every case.

## Test plan

The tests drive `decide()` from the `testdata/` cases. Each case names its expected reason code, so a test reads as a row: inputs in, one code out.

- **One case per reason code** that `decide()` can set, from the stage table in the previous section. A meta-test lists the codes in `decide.py` and fails if any code has no case.
- **Every accept row** in [appendix C](C-matcher.md#accept-and-refuse-examples), plus a real `pins` group diff copied from a merged Renovate pull request in tbhb-dev/rules.
- **Every refuse row** in appendix C.
- **Ordering.** A diff with two refusals returns the first by path and line, and `findings` holds both.
- **Fail-closed edges.** Empty inputs, a missing metadata key, a check list with an unknown status, a version with a suffix, a hunk header that doesn't parse, a CRLF line, and a 40-hex value in uppercase all refuse.
- **Adversarial diffs.** A `uses:` line with a trailing comment after the version, a second `@` in the path, a Unicode lookalike in the owner, and tab indentation all refuse.
- **Property test.** A generator mutates accepted fixtures by one byte outside the value slots, and every mutant must refuse. It uses a fixed seed, so it runs the same way every time without a third-party library.

The gather step and `approve.py` get their own tests against a fake API client that replays recorded JSON responses. Those tests cover the three head reads, the dismissal path, pagination, and the error mapping in the failure table. Nothing in the test suite calls GitHub.
