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

The matcher is the part of `decide()` that reads the diff. It answers one question: does every changed line replace one pinned value with another, and nothing else? Any doubt is a refusal.

## Input

The gather step hands the matcher three things for the pull request at its head, H:

- The output of `git diff --unified=0 -M --no-ext-diff --no-textconv --full-index` from the merge base to H, as text. Zero context lines mean every line in a hunk body is a change.
- The base and head text of each changed file that a pin kind needs to read whole: `MODULE.bazel`, listed `.bzl` files, `mise.toml`, and the two lockfiles.
- The pin facts the gather step looked up, keyed by file and line: which commit a tag names, whether its source's default branch contains a commit, and the SHA-256 hash, from the Secure Hash Algorithm (SHA) family, of each downloaded artifact.

The matcher makes no network calls. A fact it needs and doesn't have is a refusal, `fact_missing`.

## File-level rules

The matcher first reads each file's extended header lines in the diff. It refuses the whole pull request at the first file that fails, and still records every finding.

| Header in the diff | Reason code |
| --- | --- |
| Path matches no pin glob | `not_a_pin_file` |
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

Each hunk header reads `@@ -a,b +c,d @@`. A pin-only hunk has `b` equal to `d`, both at least 1. The matcher pairs the `i`-th removed line with the `i`-th added line.

| Condition | Reason code |
| --- | --- |
| `b` is 0, a pure addition | `line_added` |
| `d` is 0, a pure removal | `line_removed` |
| `b` differs from `d` | `line_count_changed` |
| A line in a pair matches no pin grammar for its file | `not_a_pin_line` |
| The two lines of a pair match different grammars | `mixed_line_kinds` |
| The pairs of one hunk fall in two different stanzas or line kinds | `hunk_spans_kinds` |
| A pair's lines differ outside the value slots of their grammar | `key_changed` |
| A pair's lines are identical in every value slot | `no_value_change` |

The last two rules carry most of the weight. Each grammar names its value slots. The matcher blanks those slots in both lines, and the two templates must match byte for byte, so indentation, quoting, keys, names, and comments outside the slots can't change.

## Workflow `uses:` lines

**Files:** `.github/workflows/*.yml`, `.github/workflows/*.yaml`, and `.github/actions/*/action.yml` or `action.yaml`, unless the caller narrows them.

**Grammar:** an optional indent, an optional list dash and space, then `uses:` and one space, and an `owner/repo` reference with an optional path, `@`, 40 lowercase hexadecimal characters, two spaces, a `#` and one space, and a version comment. The comment is a version such as `v7.0.1` or `7.0.1`, or the word `main`.

**Value slots:** the commit and the version comment.

The matcher accepts a pair when:

- The commit changed. A changed comment with the same commit is `comment_only_change`.
- For a version comment, the new version is higher than the old one, and its bump level is in `allowed_bumps`. A lower version is `downgrade`, an equal version with a new commit is `digest_without_version_change`, and a disallowed level is `bump_level_not_allowed`.
- For a version comment, the pin facts show that the tag named in the new comment points at the new commit in `owner/repo`. Anything else is `tag_mismatch`. This also defeats a commit that exists only in a fork but resolves through the parent's object network.
- For a `main` comment, `owner` is in `first_party_owners`, and the facts show the old commit is an ancestor of the new one, and the new one is an ancestor of the source's default branch head. Failures are `not_first_party`, `downgrade`, and `not_merged_at_source`. The bump level is `digest`.

Refused shapes include a `docker://` reference, a local `./` action, a tag or branch without a commit, a pin without a comment (`unversioned_pin`), a changed action name or path (`key_changed`), and any change to a `with:` input, `runs-on:`, or `name:` line (`not_a_pin_line`).

## Starlark files

**Files:** `MODULE.bazel`, plus any `.bzl` files the caller lists, such as a rules repository's `pins/*.bzl`.

The matcher reads the head and base text to find the stanza around each changed line. It walks up from the line to the nearest line at column 0 that opens one of the known calls and down to the `)` at column 0 that closes it. The stanza must open and close the same way on both sides, with the same call name and the same `name` or `module_name` value. Anything else, including a nested bracket that spans lines, a `+` concatenation on a changed line, a variable in a value slot, or a backslash continuation, is `parse_doubt`.

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

The matcher reads lockfiles as parsed documents, with `tomllib` and `json` from the standard library, and compares base to head. Every changed line in the diff must still fall inside an entry the comparison allows.

### `mise.lock`

- The `lockfile_version`, the set of tools, each tool's `backend`, and each tool's set of platform tables stay the same.
- A tool's entry may change only if its `mise.toml` version changed in this pull request. Anything else is `lock_changed_without_pin`.
- Within a changed tool, `version` and `specifiers` equal the new `mise.toml` version. Each platform's `url` changes only by the version token, `url_api` keeps its owner and repository, and `checksum` equals the SHA-256 of the downloaded new `url`. Failures are `lock_version_mismatch`, `url_changed_beyond_version`, `lock_key_changed`, and `digest_mismatch`.
- Every other key, such as `provenance` and `checksum_source`, stays the same, or the run refuses with `lock_key_changed`.

### `MODULE.bazel.lock`

- The matcher accepts a change only when a `bazel_dep` version moved in this pull request. Anything else is `lock_changed_without_pin`.
- `lockFileVersion` and `selectedYankedVersions` stay the same.
- Changes fall only under `registryFileHashes`. Each added or changed key is a URL under the registry the caller names, which defaults to the Bazel Central Registry, and each value is 64 hexadecimal characters or null.
- A change under `moduleExtensions` or any other key is `lock_key_changed`. Extension results can carry their own URLs and digests, and this release doesn't judge them.

CI runs Bazel with `--lockfile_mode=error`, which fails when the lockfile disagrees with `MODULE.bazel` ([appendix A](A-evidence.md)). The required checks therefore cover the lock's consistency, and the matcher covers its shape.

## Bump levels

The matcher compares versions as dotted integers after an optional `v`, up to three parts. A pre-release or build suffix on either side is `unparsed_version`. The level is `major` when the first part rises, `minor` for the second, and `patch` for the third. A first-party commit pin is `digest`. A major bump from `0.x` to `1.x` counts as major.

## Accept and refuse examples

Each line shows the removed line, then the added line. This table shortens hashes, and the tests use full ones.

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
| MODULE | `bazel_dep(name = "bazel_env.bzl", version = "0.10.0", …)` then `"0.10.1"` | accept, patch, if the lock change passes |
| MODULE | `bazel_dep(name = "a", …)` then `bazel_dep(name = "b", …)` | `key_changed` |
| MODULE | a new `bazel_dep` line | `line_added` |
| MODULE | `commit = "a05c…",` then `commit = "b51a…",` in the `tbhb_rules` `git_override` | accept, digest, if merged at source |
| MODULE | `remote = "https://github.com/tbhb-dev/rules.git",` changed | `not_a_pin_line` |
| MODULE | `url = ".../v0.7.2/Google.zip"` and `sha256` both changed to `v0.7.3` | accept, patch, if the download hashes to the new digest |
| MODULE | `url` changed to `v0.7.3`, `sha256` unchanged | `digest_not_updated` |
| MODULE | `sha256` changed, `url` unchanged | `digest_without_version_change` |
| MODULE | `url` changed from one host to another | `url_changed_beyond_version` |
| mise.toml | `prek = "0.5.11"` then `prek = "0.5.12"`, with its `mise.lock` entry | accept, patch |
| mise.toml | `uv = "0.13.9"` then `uv = "latest"` | `unpinned_version` |
| mise.toml | a change under `[settings]` | `not_a_pin_line` |
| mise.lock | a tool's `checksum` changed with no `mise.toml` change | `lock_changed_without_pin` |
| any | `.github/renovate.json5` preset pin changed | `not_a_pin_file`, until [D4](decisions.md) |
| any | `ci.yml` renamed to `ci.yaml` | `rename` |
| any | a workflow file's mode changed to executable | `mode_change` |
| any | a new workflow file | `new_file` |
| any | a binary file changed | `binary` |
| MODULE | one hunk that changes a `bazel_dep` version and the adjacent `http_file` digest | `hunk_spans_kinds` |
