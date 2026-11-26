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

- **The request.** `ssh_serve.py` accepts only envelope `v` 1, and `decisions/request.py` accepts only request `v` 2, the constant `REQUEST_VERSION` in `model.py`. Each refuses any other value and any unknown field. The driver also stages the request on the host as `<run>.request.json` beside the run's other files, which the run host reads back (`REQUEST` in `decisions/runfiles_names.py` and `run_host.py`, existing at tools `1be1910`).
- **The run metadata.** The run files that the host writes and the driver reads, each `<run>.<suffix>` in the runs directory and named in `decisions/runfiles_names.py`, carry no version at all.
- **The build check.** The driver runs `tbhbdev --version` on the host in its preflight and calls `same_version` in `decisions/driver.py`, which refuses any build that differs from its own. tools#400 adds one sync and a retry of the same exact match, so a mismatch that survives the sync still refuses.

So every release has to reach every machine at once. `run_host.py` and `decisions/argv.py` took 47 commits between 2026-11-01 and 2026-11-25, so releases are frequent, and a frequent lockstep release refuses runs often.

## The protocol schema

One file, `schemas/agent-protocol.json`, describes everything the driver and the host exchange (D16). It is a JSON Schema document with a top-level integer `version`, and it defines three parts:

- **`envelope`**, the frame `ssh_serve` reads, with its `push_token`. Its own `v` stays 1 as a frame marker, and the schema pins it.
- **`request`**, the run request. Its `v` is the protocol version.
- **`metadata`**, one definition per run file the driver or `agent status` reads, such as `.phase`, `.rc`, `.audit.json`, and an `index.jsonl` line. A plain-text file is checked as a JSON string against its pattern.

The protocol version is the schema's `version`, and nothing else names a protocol (D14). It starts at 2, today's request version, so no machine sees a new number at the switch. The separate plugin version contract was replaced by the 2026-11-26 no-plugins decision (D3).

### Both sides validate

The Python package ships the file as package data and validates with `jsonschema`, already a tools dependency (existing). The phase 1 release adds the schema and its CI job in tools, and both move with the Python CLI on import. The Go code embeds a copy in `factory/cli/internal/agent/protocol` with `go:embed` and validates in-process through `cuelang.org/go`, never a `cue` child process (D20). A monorepo test compares that copy with the local source file byte for byte.

- **Writers are strict.** The driver validates each request, and the host each metadata file, before writing it, against the schema with `additionalProperties: false`. A code change that adds a field fails its own tests until the schema names the field.
- **Readers are lenient.** A reader removes properties the schema does not name, logs their names as a warning, and then validates. That is what lets an additive change reach an older reader without a refusal.
- **Validation is pure.** It takes bytes and a schema and returns errors, so it lives in the pure packages `decisions/protocol.py` and `internal/agent/protocol`.

### The legacy version 2 shape

Hosts installed before phase 1 refuse every unknown field. Phase 1 therefore freezes today's exact request and envelope as `schemas/agent-protocol.v2-legacy.json`, with the fields `decisions/request.py` accepts at the phase 1 commit and nothing else.

- **Who gets it.** A host that does not answer the protocol query is a legacy host. The driver sends it version 2 in the legacy shape.
- **How a writer makes it.** The writer builds the request in the chosen version, drops every property the legacy file lacks, and validates the result against the legacy file before it sends anything. Additive properties are those an older reader may drop, so dropping them keeps the run's meaning. A property the legacy file lacks that the run cannot drop is breaking by the table below and never reaches version 2.
- **The staged file.** The driver writes `<run>.request.json` from the same projected bytes as the envelope's request, so a legacy run host never reads a field it would refuse. A test compares the two byte for byte for each corpus member.
- **After the barrier.** The rollout barrier leaves no legacy host on the fleet list, so the legacy shape is a safety net for a host that answers no protocol query.
- **How long it lasts.** A build writes the legacy shape while it serves version 2 for new launches, which is through the build that bumps to version 3, since that build serves 3 and 2. From the version 4 bump on, a driver reaches version 2 only through a bridge. The legacy file itself is a retained schema and stays for good, so a reader can still check a run from before phase 1.

### Run files carry their version

A run's files outlive the build that made them. A run started on one build is read by `agent status`, `agent wait`, and the driver after either side updates.

- **The version file.** The host writes `<run>.protocol`, the chosen version as a decimal integer, as the run's first file, before `<run>.phase`. The schema's `metadata` part defines it.
- **Readers follow it.** Every metadata reader reads `<run>.protocol` first and validates the run's other files against that version's frozen schema, not against the reader's own version. A missing `<run>.protocol` means a run from before phase 1, read as the legacy shape.
- **Reading keeps every version.** Each build serves two versions for new launches, but it retains every frozen schema file, so it can read any run still on disk. The files are small, and a test reads one run's files per retained version.

### Additive and breaking changes

| Change | Kind | Version |
| --- | --- | --- |
| A new optional property whose absence keeps today's behavior, and which an older reader may drop without widening what a run may do | additive | kept |
| A new definition that only newer readers read | additive | kept |
| Removing or renaming a property, making one required, changing a type, or narrowing an enum or pattern | breaking | bumped by 1 |
| A new property that an older host must not ignore, such as a new limit or sandbox rule | breaking | bumped by 1 |

On a bump, the previous file moves to `schemas/agent-protocol.v<N>.json` and freezes. Two sets of schemas then exist, and each has its own job:

| Set | What it holds | What uses it | What CI holds |
| --- | --- | --- | --- |
| Served | The head schema and the newest frozen file, the build's own version and the previous one | Writers of new requests and run files, and hosts choosing a version | The writer and reader corpus, one directory per served version, `tests/agent-corpus/v<N>/` |
| Retained | Every frozen file from `v2` up, and the legacy version 2 file | Readers of runs already on disk, at the version each run's `<run>.protocol` names | A reader fixture of one run's files per retained version, under `tests/agent-runs/v<N>/` |

So the host keeps serving the previous version for new launches until the next bump drops it from the served set, and every build keeps reading it for good. A frozen file never changes and is never deleted.

A breaking bump merges only when the fleet check shows every listed machine on a build that serves the version being replaced. The fleet check is `tbhbdev alpha version-check` (existing), which phase 1 extends to print each machine's served versions, until wave 4 ports it as `tbhbdev install check`. That keeps every installed build within one version of every other, so any driver and host share a version.

### Choosing a version per launch

The preflight script prints the host's build line, then the versions it serves from `tbhbdev agent protocol`. Every Python build from the phase 1 release on answers the top-level name `agent` with the `alpha agent` code, `agent protocol` included, so drivers in both languages ask by the final name and no Go code names `alpha` (D5). A host too old to answer serves only the legacy version 2 shape.

`protocol.Choose(driver, host)` is pure. It takes each side's whole served set, its own two versions and every version its bridges serve, and returns the highest version in both, or `NoCommon` with both sets. The driver sends the chosen version and logs it. A build mismatch logs one warning line with both builds and never stops the launch (D14).

### Bridges

A bridge is an older bundle that an install keeps so it can still speak an older version. It is what lets a launch go on when an update fails.

- **Kept on every install.** Beside the link's target, each install keeps the newest bundle that serves each protocol version from 2 up. The release job publishes a pointer for each version, `protocol-v<N>.json`, naming the newest main bundle that serves it. Before it moves the link, the installer fetches and checks any missing bridge under D23, and an install without its bridges does not publish. Pruning never removes a bridge ([appendix J](J-self-update.md#runs-in-progress)), so a launch never needs the network to reach one.
- **Driving through a bridge.** When the chosen version is one the driver's own build does not write, the driver execs the bridge that serves it, with `TBHBDEV_NO_SELF_UPDATE=1` and the original argv, and that build drives the launch.
- **Serving through a bridge.** When the envelope's version is one the host's own build does not serve, the host's server hands the envelope to the bridge that serves it, the way the phase 3 Go server hands what it cannot parse to `tbhbdev-py`: the bytes already read, then the rest of standard input, with the descriptors passed and the child's status as its own.
- **Retired by the fleet check.** A bridge goes only when the fleet check shows that no listed machine serves only its version or older.

Every bundle carries bridges back to version 2, and every Python build from phase 1 on serves version 2, so any two listed machines share a version. A launch that goes through a bridge still tries to close the gap first, once:

1. **Update the older side.** When the host is older, preflight runs the host's front with `TBHBDEV_UPDATE_NOW=1`, which makes `Decide` refresh at once ([appendix J](J-self-update.md#the-decision)). When the driver is older, it runs its own self-update the same way and execs itself again.
2. **Choose again.** The driver reads the host's versions again and calls `Choose`.
3. **Launch.** The launch goes on the chosen version, through a bridge if one is still needed. A failed or slow update only adds a warning line that names both version sets and the machine to update, which the ops maintainer gets as a broken invariant, since the bump gate exists to make bridges rare.

`NoCommon` is left only for an install that has lost a bridge, a damaged install and not a version mismatch. The driver then runs the installer's bridge fetch and chooses again, and it exits 3 only when that fetch fails too, naming the damaged install for the ops maintainer.

Table tests cover each `Choose` row: equal sets, one-version skew each way, a legacy host, a version served only by a bridge on either side, and no common version. An `io_go_test` launches between a driver and a host two breaking versions apart, with the update made to fail, and the launch runs through the bridge.

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
- `kind = "breaking"` and `version` is not the base's plus 1, the head does not add `agent-protocol.v<base>.json` matching the base file byte for byte, or `fleet` is empty.
- The `version` changes with no change to the schema.
- Any retained file, a frozen `v<N>.json` or the legacy version 2 file, changes or is deleted, or a frozen file appears other than the one a breaking entry adds.
- The served set and the corpus disagree: a served version lacks its `tests/agent-corpus/v<N>/` directory, or a retained version lacks its `tests/agent-runs/v<N>/` fixture. A corpus directory for a version that left the served set moves to the reader fixtures in the bump's pull request.

The diff cannot judge whether a new property is safe to ignore, so the entry's `why` states it and the reviewer checks it. The job's own rules ship with planted failing fixtures, as every new check does ([appendix F](F-build-and-checks.md#checks)).

## The transition

At no phase does a build mismatch refuse a launch, and at no phase does a host lack a version the driver can send. The one refusal left is the check in builds installed before phase 1. The phase 1 rollout barrier removes those builds from the fleet before launches resume, by sync or, for a machine the sync cannot reach, by fencing it off the list until a sync does.

| Phase | Machines run | Python driver, Go host | Go driver, Python host | Mixed bundle versions |
| --- | --- | --- | --- | --- |
| 1 | Python, installed as today, behind the rollout barrier | none yet | none yet | Not bundles yet: phase 1 drivers log a build mismatch and send version 2 |
| 2 | Phase 1 Python, or a bundle with `tbhbdev-py` behind the front | none yet | none yet | Each runs the Python from its own release, chooses a version, and logs a build mismatch |
| 3 | Bundles, with Go `agent` and Python still bundled | The Go server serves both versions its build serves, and passes any request it cannot parse to the bundled `tbhbdev-py` server | The Python host answers the protocol query, and the Go driver picks the common version | Every build serves its version and the one before, the bump gate keeps skew to one, and bridges cover any wider skew |
| 4 | Bundles, Python server removed | none left | none left | As phase 3 |

### Phase 1: the release that removes the refusal

A Python release in tools replaces `same_version` and tools#400's `version_heal`, which reinstalls the host once and retries the same exact match, with the warning, adds the top-level `agent` name with `agent protocol`, its `alpha agent protocol` form, and the `<run>.protocol` file, ships the schema and the legacy shape and validates against them, and makes readers drop unknown properties. Preflight always runs its `git ls-remote`, where today a mismatch makes it exit early.

Builds already installed still refuse any host whose build differs, and no release can change code that is already installed. An install order cannot fix that, because a sync-once retry from an old driver still ends in the exact match. So phase 1 ends with a rollout barrier, run once by the ops maintainer:

1. **Pause.** Launches stop. The coordination loop and every session that drives launches hold new `agent exec` calls, and runs already started finish.
2. **Sync.** The ops maintainer syncs every listed machine to the phase 1 release, or any later build, in one event that tbhb starts.
3. **Fence.** A machine the sync could not reach leaves the fleet list. The ops maintainer removes it from the hosts that drivers launch on, stops the sessions that drive launches from it, and removes its driver key from every host's authorized keys, so it is neither a driver nor a host.
4. **Check and resume.** `tbhbdev alpha version-check` shows every listed machine on phase 1 or later and none older. The barrier record lists each machine and its build, and each fenced machine. Launches then resume.

With launches resumed, no listed machine carries the old check, so D6 holds across the fleet with no exception. A fenced machine rejoins only through a sync the ops maintainer runs, then a version check that shows it on phase 1 or later, and only then do its key and its list entry come back. The same rejoin rule holds for a machine restored from an old image.

### Phase 3: the Go server

After callers move, a host's forced command becomes `tbhbdev agent ssh-serve`. The Go server validates against the same schema bytes as Python, so both accept the same requests. As a safety net while Python is still bundled, a request the Go server cannot parse goes to the bundled `tbhbdev-py` server.

The Go server starts that server as a child, feeds it the bytes already read followed by the rest of standard input, passes the other descriptors, forwards SIGINT and SIGTERM, and exits with the child's status.

CI runs the corpus four ways: the Python and Go servers each against every envelope of each served version, and each driver's output checked against a corpus member. Changes to the Python launcher after wave 2 land in Go first, so the two cannot drift.

### Phase 4: the Python server retires

When the fleet check shows every host on a bundle with the forced command on Go, and the forwarder logs show no `alpha agent` calls for 14 days, the Python server goes. The monorepo keeps the schema, its decisions file, and the `protocol-version` job. Versions change only under the schema rules in this appendix.

Self-update ([appendix J](J-self-update.md)) and event-driven sync keep the fleet near main, so skew is usually zero and the previous version is rarely used.
