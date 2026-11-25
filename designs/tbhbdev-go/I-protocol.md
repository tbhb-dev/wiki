---
type: Design Appendix
title: tbhbdev Go port protocol and versions
description: The versioned schema that the agent driver and host share, how its version changes, the CI check that holds it, and the transition in which no launch is refused.
tags: [tbhbdev, go, protocol, versions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

<!-- cspell:words jsonschema runfiles sigterm cuelang -->

# tbhbdev Go port protocol and versions

Back to [the design](README.md).

Everything here is new unless it names an existing file. [Appendix D](D-evidence.md#the-agent-protocol-today) cites what exists today.

## Today

The driver and the host share no schema file and negotiate no version (existing).

- **The request.** `ssh_serve.py` accepts only envelope `v` 1, and `decisions/request.py` accepts only request `v` 2, the constant `REQUEST_VERSION` in `model.py`. Each refuses any other value and any unknown field.
- **The run metadata.** The run directory files that the host writes and the driver reads, named in `decisions/runfiles_names.py`, carry no version at all.
- **The build check.** The driver runs `tbhbdev --version` on the host in its preflight and calls `same_version` in `decisions/driver.py`, which refuses any build that differs from its own.

So every release has to reach every machine at once. `run_host.py` and `decisions/argv.py` took 47 commits between 2026-11-01 and 2026-11-25, so releases are frequent, and a frequent lockstep release refuses runs often.

## The protocol schema

One file in tools, `schemas/agent-protocol.json`, describes everything the driver and the host exchange (D16). It's a JSON Schema document with a top-level integer `version`, and it defines three parts:

- **`envelope`**, the frame `ssh_serve` reads, with its `push_token`. Its own `v` stays 1 as a frame marker, and the schema pins it.
- **`request`**, the run request. Its `v` is the protocol version.
- **`metadata`**, one definition per run directory file the driver or `agent status` reads, such as `.phase`, `.rc`, `.audit.json`, and an `index.jsonl` line. A plain-text file is checked as a JSON string against its pattern.

The protocol version is the schema's `version`, and nothing else names a protocol (D14). It starts at 2, today's request version, so no machine sees a new number at the switch.

### Both sides validate

The Python package ships the file as package data and validates with `jsonschema`, already a tools dependency (existing). The Go packages embed the same bytes with `go:embed` and validate in-process through `cuelang.org/go`, never a `cue` child process (D20), and a build test fails if the two copies differ.

- **Writers are strict.** The driver validates each request, and the host each metadata file, before writing it, against the schema with `additionalProperties: false`. A code change that adds a field fails its own tests until the schema names the field.
- **Readers are lenient.** A reader removes properties the schema doesn't name, logs their names as a warning, and then validates. That's what lets an additive change reach an older reader without a refusal.
- **Validation is pure.** It takes bytes and a schema and returns errors, so it lives in the pure packages `decisions/protocol.py` and `internal/agent/protocol`.

### Additive and breaking changes

| Change | Kind | Version |
| --- | --- | --- |
| A new optional property whose absence keeps today's behavior, and which an older reader may drop without widening what a run may do | additive | kept |
| A new definition that only newer readers read | additive | kept |
| Removing or renaming a property, making one required, changing a type, or narrowing an enum or pattern | breaking | bumped by 1 |
| A new property that an older host must not ignore, such as a new limit or sandbox rule | breaking | bumped by 1 |

On a bump, the previous file moves to `schemas/agent-protocol.v<N>.json` and freezes. Each build serves its own version and the previous one, as a driver and as a host, so the host keeps serving the previous version until the next bump retires it. The corpus keeps one directory per served version, `tests/agent-corpus/v<N>/`.

A breaking bump merges only when `tbhbdev install check` shows every listed machine on a build that serves the version being replaced. That keeps every installed build within one version of every other, so any driver and host share a version.

### Choosing a version per launch

The preflight script prints the host's build line, then the versions it serves from `tbhbdev agent protocol`, or `alpha agent protocol` on a Python host. A host too old to answer counts as serving 2.

`protocol.Choose(driver, host)` is pure and returns the highest version both serve. The driver sends that version and logs it. A build mismatch logs one warning line with both builds and never stops the launch (D14).

## The schema version check

A planned CI job in tools, `protocol-version`, runs on `pull_request` and `merge_group` events. It needs the merge base, so it runs outside Bazel, and its comparison is a pure function with table tests.

It compares `schemas/agent-protocol.json` at the merge base with the file at the head. When the bytes differ, it reads the head's `schemas/agent-protocol.decisions.toml` and fails unless one entry has a `schema_sha256` equal to the head file's Secure Hash Algorithm 256 (SHA-256) hash. That entry is the version decision, reviewed in the same pull request:

```toml
[[change]]
schema_sha256 = "<hex of the head schema>"
version = 3
kind = "breaking"
why = "request.memory_mib becomes required"
fleet = "install check, 2026-12-02: every listed machine serves 2"
```

The job also fails when any of these holds:

- `kind = "additive"` and `version` differs from the base's, or a structural diff of the two schemas finds a breaking edit from the preceding table.
- `kind = "breaking"` and `version` isn't the base's plus 1, `agent-protocol.v<base>.json` doesn't match the base file byte for byte, the older frozen file remains, or `fleet` is empty.
- The `version` changes with no change to the schema, or a frozen `v<N>.json` file changes.

The diff can't judge whether a new property is safe to ignore, so the entry's `why` states it and the reviewer checks it. The job's own rules ship with planted failing fixtures, as every new check does ([appendix F](F-build-and-checks.md#checks)).

## The transition

At no phase does a build mismatch refuse a launch, and at no phase does a host lack a version the driver can send. The one refusal left is the check in builds installed before phase 1, and phase 1's install order keeps them from meeting a newer host.

| Phase | Machines run | Python driver, Go host | Go driver, Python host | Mixed bundle versions |
| --- | --- | --- | --- | --- |
| 1 | Python, installed as today | none yet | none yet | Not bundles yet: phase 1 drivers log a build mismatch and send version 2 |
| 2 | Phase 1 Python, or a bundle with Python behind the front | none yet | none yet | Both run Python from their own commits and choose a version, and a build mismatch logs |
| 3 | Bundles, with Go `agent` and Python still bundled | The Go server serves both versions its build serves, and passes any request it can't parse to the bundled Python server | The Python host answers the protocol query, and the Go driver picks the common version | Every build serves its version and the one before, and the bump gate keeps skew to one |
| 4 | Bundles, Python server removed | none left | none left | As phase 3 |

### Phase 1: the release that removes the refusal

A Python release replaces `same_version` with the warning, adds `alpha agent protocol`, ships the schema and validates against it, and makes readers drop unknown properties. Preflight always runs its `git ls-remote`, where today a mismatch makes it exit early.

Builds already installed still refuse any host whose build differs, and no new release can change code that's already installed. Phase 1 therefore installs on every machine that drives launches before any machine that only hosts them. A phase 1 driver accepts every host, and every host before it serves version 2.

Which machines drive is unverified, and plan step 2 lists them. If more than one drives, they install in one sync event, so the window in which two old drivers could meet a new host is that event's length.

### Phase 3: the Go server

After callers move, a host's forced command becomes `tbhbdev agent ssh-serve`. The Go server validates against the same schema bytes as Python, so both accept the same requests. As a safety net while Python is still bundled, a request the Go server can't parse goes to the bundled Python server.

The Go server starts that server as a child, feeds it the bytes already read followed by the rest of standard input, passes the other descriptors, forwards SIGINT and SIGTERM, and exits with the child's status.

CI runs the corpus four ways: the Python and Go servers each against every envelope of each served version, and each driver's output checked against a corpus member. Changes to the Python launcher after wave 2 land in Go first, so the two can't drift.

### Phase 4: the Python server retires

When `tbhbdev install check` shows every host on a bundle with the forced command on Go, and the forwarder logs show no `alpha agent` calls for 14 days, the Python server goes. Versions then change only under the schema rules in this appendix.

Self-update ([appendix J](J-self-update.md)) and event-driven sync keep the fleet near main, so skew is usually zero and the previous version is rarely used.
