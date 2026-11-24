---
type: Design Appendix
title: Renovate auto-approval pin-file matcher
description: The grammar for each pin file kind, the refusals, and the accept and refuse examples for the Renovate auto-approval matcher.
tags: [renovate, approvals, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Renovate auto-approval pin-file matcher

Back to [the design](README.md).

The matcher is the part of `decide()` that reads the diff. It accepts only pin replacements and the companion lockfile changes defined below. Any doubt is a refusal.

## Input

The gather step hands the matcher three things for the pull request at its head, H:

- The output of `git diff --unified=0 -M --no-ext-diff --no-textconv --full-index` from the merge base to H, plus the old and new file modes from `git diff --raw` over the same commits. Zero context lines mean every line in a hunk body is a change.
- The base and head text of each changed pin file.
- Pin facts from the gather step, indexed by file and line number. These include tag targets and commit ancestry. For downloads, gather the Secure Hash Algorithm (SHA-256) digest and the release asset metadata needed to match the two URLs in `mise.lock`.

The matcher makes no network calls. A fact it needs and doesn't have is a refusal, `fact_missing`.

## File-level rules

The matcher first reads each file's extended header lines in the diff. It requires both file modes as evidence and refuses with `parse_doubt` if it lacks either mode for the file. It refuses the whole pull request at the first file that fails, and still records every finding.

| Header in the diff | Reason code |
| --- | --- |
| Path matches no pin glob | `not_a_pin_file` |
| Path is the reusable approver workflow file | `self_pin` |
| `rename from` or `similarity index` | `rename` |
| `copy from` | `copy` |
| `new file mode` | `new_file` |
| `deleted file mode` | `deleted_file` |
| `old mode` and `new mode` | `mode_change` |
| `Binary files` or `GIT binary patch` | `binary` |
| Mode `160000` or `120000` on either side | `submodule_or_symlink` |
| A `\ No newline at end of file` marker | `eol_change` |
| Bytes that aren't valid UTF-8, or a carriage return | `encoding` |

A pull request with no changed lines gets `empty_diff`.

## Hunk-level rules

Git writes the old and new ranges as `@@ -a,b +c,d @@` and may append a section heading. An omitted count on either side means 1, as in `@@ -1 +1 @@`. The matcher checks the body against both counts.

For text pins, the old and new counts must match and be at least 1. Pair the `i`-th removed line with the `i`-th added line, and find the stanza or field containing each line in its own file version. Each pair must belong to the same stanza or field on both sides, even when the hunk covers more than one stanza or pin kind.

| Condition | Reason code |
| --- | --- |
| `b` is 0, a pure addition | `line_added` |
| `d` is 0, a pure removal | `line_removed` |
| `b` differs from `d` | `line_count_changed` |
| A line in a pair matches no pin grammar for its file | `not_a_pin_line` |
| The two lines of a pair match different grammars | `mixed_line_kinds` |
| A pair's lines differ outside the value slots of their grammar | `key_changed` |
| A pair's lines are identical in every value slot | `no_value_change` |

The last two rules carry most of the weight. Each grammar names its value slots. The matcher blanks those slots in both lines, and the two templates must match byte for byte, so indentation, quoting, keys, names, and comments outside the slots can't change.

Lockfile lines use the parsed-entry rules below. After validating the hunk header and body, the matcher allows unpaired lines that add registry entries under the `MODULE.bazel.lock` rules, including the JSON delimiter changes needed for insertion. Other unpaired lines still trigger the addition, removal, or count refusals.

## Workflow `uses:` lines

**Files:** `.github/workflows/*.yml`, `.github/workflows/*.yaml`, and `.github/actions/*/action.yml` or `action.yaml`.

The matcher refuses even pin-only changes to `.github/workflows/renovate-auto-approval.yml`, the reusable approver. A pin to the reusable workflow in a separate caller can qualify under [D11](decisions.md).

**Grammar:** an optional indent, an optional list dash and space, then `uses:` and one space, and an `owner/repo` reference with an optional path, `@`, 40 lowercase hexadecimal characters, two spaces, a `#` and one space, and a version comment. The comment is a version such as `v7.0.1` or `7.0.1`, or the word `main`.

A conservative block parser must identify the line as a `uses` field of a workflow job or an action step in both YAML documents. Text inside a block scalar fails with `not_a_pin_line`, and ambiguous syntax fails with `parse_doubt`.

**Value slots:** the commit and the version comment.

The matcher accepts a pair when:

- The commit changed. A changed comment with the same commit is `comment_only_change`.
- For a version comment, the new version is higher than the old one, and its bump level is in `allowed_bumps`. A lower version is `downgrade`, an equal version with a new commit is `digest_without_version_change`, and a disallowed level is `bump_level_not_allowed`.
- For a version comment, the pin facts show that the tag named in the new comment points at the new commit in `owner/repo`. Anything else is `tag_mismatch`. This also defeats a commit that exists only in a fork but resolves through the parent's object network.
- For a `main` comment, `owner` is in `first_party_owners`, and the facts show the old commit is an ancestor of the new one, and the new one is an ancestor of the source's default branch head. Failures are `not_first_party`, `downgrade`, and `not_merged_at_source`. The bump level is `digest`.
- A tag pin from `first_party_owners` must also pass the merged-at-source check for the new commit. [D2](decisions.md) governs approval for both branch and tag pins.

Refused shapes include a `docker://` reference, a local `./` action, a tag or branch without a commit, a pin without a comment (`unversioned_pin`), a changed action name or path (`key_changed`), and any change to a `with:` input, `runs-on:`, or `name:` line (`not_a_pin_line`).

## Starlark files

**Files:** `MODULE.bazel` and `pins/*.bzl`.

The matcher scans both base and head text for known calls at any space indentation, including inside a function. It pairs each opening parenthesis with its closing parenthesis by tracking delimiters outside strings and comments, so a call can span lines or occupy one line. It maps each changed line to a unique containing call in its own file version.

For each line pair, the calls on both sides must occupy the same position in the file's stanza order. Compare their call names and their `name` or `module_name` values, and require matching opening and closing forms. Ambiguous boundaries, a nested bracket spanning lines within a pin call, a `+` concatenation on a changed line, a variable in a value slot, and a backslash continuation all give `parse_doubt`.

The matcher gathers all changed lines for each stanza before checking a URL version against its digest or applying other rules that use more than one field.

### `bazel_dep`

**Grammar:** a single-line `bazel_dep(name = "x", version = "1.2.3", ...)`, or a `version = "1.2.3",` line inside a multi-line `bazel_dep`.

**Value slot:** the version string. The new version is higher, and `allowed_bumps` lists its level. A `bazel_dep` change also needs the matching `MODULE.bazel.lock` change, or the run refuses with `lock_not_updated`.

### `git_override`, the shared-rules pin

**Grammar:** a `commit = "<40 hex>",` line inside a `git_override` stanza.

**Value slot:** the commit. The stanza's `remote` names a repository under one of `first_party_owners`, and its `module_name` matches `first_party_modules`, which defaults to `tbhb_rules*`. The ancestry facts are the same as a `main` `uses:` pin. Failures are `not_first_party`, `downgrade`, and `not_merged_at_source`.

The shared-rules pin moves in the same pull request as the `tbhb-dev/rules` `uses:` pins. The matcher checks each line on its own, and each must pass.

### `http_archive`, `http_file`, and `archive_override`

**Grammar:** inside one of these stanzas, a `url = "..."` line, a one-line `urls = ["..."]` line, a `strip_prefix = "..."` line, a `sha256 = "<64 hex>"` line, or an `integrity = "sha256-<base64>"` line.

**Value slots:** the version token in each URL and `strip_prefix`, and the whole digest.

The matcher accepts the stanza when:

- Every URL and `strip_prefix` changes by replacing one version token with another, everywhere it appears, and nothing else. Anything more is `url_changed_beyond_version`.
- The digest changes in the same stanza. A version change with the old digest is `digest_not_updated`. A digest change with no version change is `digest_without_version_change`.
- The new digest equals the SHA-256 the gather step computed for the downloaded new URL. A mismatch is `digest_mismatch`, and a download that failed or ran over `max_download_mb` is `digest_unverifiable`.
- The version rises within the allowed levels. A commit hash used as a version token counts as `digest` and needs a first-party source, as for `git_override`.

A URL held in a variable, such as a module-level base URL joined with `+`, is `parse_doubt`.

## `mise.toml`

**Grammar:** inside the `[tools]` table, `tool = "1.2.3"`, or `tool = { version = "1.2.3", ... }` on one line. The matcher finds the table by walking up to the nearest table header in the head and base text.

**Value slot:** the version. It must be an exact version, so `latest`, a prefix such as `1.2`, and ranges are `unpinned_version`. The new version is higher, and `allowed_bumps` lists its level. A changed line in any other table is `not_a_pin_line`. When the repository has a `mise.lock`, the tool's lock entry must change to the same version, or the run refuses with `lock_not_updated`.

## Lockfiles

The matcher compares the parsed base and head documents with `tomllib` and `json` from the standard library. It maps changed lines to the source spans of changed parsed entries on each side and requires each entry to match the manifest pin by tool and version in `mise.lock` or by module and version in the registry URL in `MODULE.bazel.lock`.

### `mise.lock`

- The `lockfile_version`, the set of tools, each tool's `backend`, and each tool's set of platform tables stay the same.
- A tool's entry may change only if its `mise.toml` version changed in this pull request. Anything else is `lock_changed_without_pin`.
- Within a changed tool, `version` and `specifiers` equal the new `mise.toml` version. Each platform's `url` changes only by the version token, `url_api` keeps its owner and repository, and `checksum` equals the SHA-256 of the downloaded new `url`. Failures are `lock_version_mismatch`, `url_changed_beyond_version`, `lock_key_changed`, and `digest_mismatch`.
- When present, `url_api` must use the form `https://api.github.com/repos/{owner}/{repo}/releases/assets/{id}` with a numeric ID. The gather step reads metadata from that endpoint, and the matcher requires the asset's `browser_download_url` to equal that platform's new `url` exactly. An invalid endpoint or a different artifact URL gives `lock_asset_mismatch`, and missing metadata gives `fact_missing`.
- Every other key, such as `provenance` and `checksum_source`, stays the same, or the run refuses with `lock_key_changed`.

### `MODULE.bazel.lock`

- The matcher accepts a change only when a `bazel_dep` version moved in this pull request. Anything else is `lock_changed_without_pin`.
- `lockFileVersion` and `selectedYankedVersions` stay the same.
- Changes fall only under `registryFileHashes`, and existing keys stay present. Each added or changed key is a URL under the registry the caller names, which defaults to the Bazel Central Registry, and each value is 64 hexadecimal characters or null. Accept standalone additions that pass these rules and match a moved pin.
- A change under `moduleExtensions` or any other key is `lock_key_changed`. Extension results can carry their own URLs and digests, and this release doesn't judge them.

CI runs Bazel with `--lockfile_mode=error`, which fails when the lockfile disagrees with `MODULE.bazel` ([appendix A](A-evidence.md)). The required checks therefore cover the lock's consistency, and the matcher covers its shape.

## Bump levels

The matcher compares versions as dotted integers after an optional `v`, up to three parts. A pre-release or build suffix on either side is `unparsed_version`. The level is `major` when the first part rises, `minor` for the second, and `patch` for the third. A first-party commit pin is `digest`. A major bump from `0.x` to `1.x` counts as major.

## Accept and refuse examples

The table describes the change, with removed lines before added lines where both appear. This table shortens hashes, and the tests use full ones.

| Kind | Removed, then added | Result |
| --- | --- | --- |
| uses | `uses: actions/checkout@3d3c…  # v7.0.1` then `@e1f2…  # v7.0.2` | accept, patch, if the tag `v7.0.2` names `e1f2…` |
| uses | `@3d3c…  # v7.0.1` then `@e1f2…  # v8.0.0` | `bump_level_not_allowed` |
| uses | `@3d3c…  # v7.0.1` then `@e1f2…  # v7.0.0` | `downgrade` |
| uses | `@3d3c…  # v7.0.1` then `@e1f2…  # v7.0.1` | `digest_without_version_change` |
| uses | `actions/checkout@…  # v7.0.1` then `evil/checkout@…  # v7.0.2` | `key_changed` |
| uses | `tbhb-dev/rules/…@a05c…  # main` then `@b51a…  # main` | accept, digest, if `b51a…` descends from `a05c…` and `main` contains it |
| uses | `@a05c…  # main` then `@b51a…  # main` from a fork commit | `not_merged_at_source` |
| uses | `uses: actions/checkout@…  # v7` and a following `with:` line changed | `not_a_pin_line` |
| uses | `runs-on: ubuntu-24.04` then `runs-on: ubuntu-26.04` | `not_a_pin_line` |
| uses | A pin moves in the reusable approver workflow file | `self_pin` |
| MODULE | `bazel_dep(name = "bazel_env.bzl", version = "0.10.0", …)` then `"0.10.1"` under `@@ -1 +1 @@` | accept, patch, if the lock change passes |
| MODULE | Two adjacent `bazel_dep` versions rise by a patch in one hunk | accept if each stanza and its lock change pass |
| MODULE | `bazel_dep(name = "a", …)` then `bazel_dep(name = "b", …)` | `key_changed` |
| MODULE | a new `bazel_dep` line | `line_added` |
| MODULE | `commit = "a05c…",` then `commit = "b51a…",` in the `tbhb_rules` `git_override` | accept, digest, if merged at source |
| MODULE | `remote = "https://github.com/tbhb-dev/rules.git",` changed | `not_a_pin_line` |
| MODULE | `url = ".../v0.7.2/Google.zip"` and `sha256` both changed to `v0.7.3` | accept, patch, if the download hashes to the new digest |
| MODULE | `url` changed to `v0.7.3`, `sha256` unchanged | `digest_not_updated` |
| MODULE | `sha256` changed, `url` unchanged | `digest_without_version_change` |
| MODULE | `url` changed from one host to another | `url_changed_beyond_version` |
| `pins/*.bzl` | An indented `http_archive` inside a function changes its URL version and digest | accept if the new digest matches the download and the bump qualifies |
| `MODULE.bazel.lock` | The diff adds a registry entry for the new version of a bumped `bazel_dep` | accept if the entry passes the lockfile rules |
| mise.toml | `prek = "0.5.11"` then `prek = "0.5.12"`, with its `mise.lock` entry | accept, patch |
| mise.toml | `uv = "0.13.9"` then `uv = "latest"` | `unpinned_version` |
| mise.toml | a change under `[settings]` | `not_a_pin_line` |
| mise.lock | a tool's `checksum` changed with no `mise.toml` change | `lock_changed_without_pin` |
| mise.lock | A version bump updates a platform's checksum and both URLs identify the same new artifact | accept if the other lockfile rules pass |
| mise.lock | A platform's `url_api` names a different release asset from its `url` | `lock_asset_mismatch` |
| any | `.github/renovate.json5` preset pin changed | `not_a_pin_file`, until [D4](decisions.md) |
| any | `ci.yml` renamed to `ci.yaml` | `rename` |
| any | a workflow file's mode changed to executable | `mode_change` |
| any | a new workflow file | `new_file` |
| any | a binary file changed | `binary` |
| MODULE | One hunk changes a `bazel_dep` version and only the adjacent `http_file` digest | `digest_without_version_change` |
