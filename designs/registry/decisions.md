---
type: Design Appendix
title: Internal container registry decisions
description: Two operator asks and the authority for registry choices.
tags: [registry, decisions, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Internal container registry decisions

Back to [the design](README.md).

## Open

### D1. Registry tag, Tailscale Service, and grants

- **Options:** dedicated registry tag, Service, and grants, or loopback-only access.
- **Recommendation:** yes, add a tag for the registry sidecar, a Tailscale Service, and least-privilege grants.
- **Why:** approved clients need tailnet access, and a new tag or grant needs the operator's recorded decision before any policy pull request merges.
- **If yes:** the network operator writes the policy with accept and deny tests, then ops deploys from merged main.
- **If no:** keep the loopback spike and direct image loads; do not merge the new policy.
- **Reversible:** yes, stop the sidecar and remove the Service and grants.
- **Evidence:** new request from the 2026-11-24 brief; current policy and available Service support remain unverified because this run did not read the policy or inspect the tailnet. Step 4 checks both before deployment.

### D2. Backup to the secondary NAS

- **Options:** a backup on the secondary NAS or a local test copy only.
- **Recommendation:** yes, let the backup operator add the job and restore drill.
- **Why:** the job touches the secondary NAS, so it needs an operator answer rather than an ops assumption.
- **If yes:** the backup operator checks destination access, adds the job from merged main, and proves a restored registry serves pulls.
- **If no:** keep the local restore check, record the missing off-host copy, and avoid treating the registry as the only image source.
- **Reversible:** yes, stop the job and revoke its access while preserving source data.
- **Evidence:** new request from the 2026-11-24 brief; destination access, capacity, and backup wiring remain unverified because this run did not inspect the secondary NAS.

The operator can answer each item in one line, such as `D1: yes`.

## Decided

D3 and D4 are recommended ops calls for the build, not claims of completed deployment. D5 through D7 carry prior authority from the supplied brief; the underlying private records remain unverified here.

### D3. Push identities

- **Options:** per-repository identities for main-branch CI jobs or a shared push identity; recommend per-repository identities.
- **Decided:** 2026-11-24 design recommendation for ops to adopt as a reversible call.
- **Rule:** grant `read`, `create`, and `update` only for the job's repository, only to main-branch continuous integration (CI) jobs. Keep push credentials separate from all pull credentials and deny routine deletion.
- **Undo:** revoke the push identities and return publishers to direct image loads.
- **Record:** the 2026-11-24 registry brief assigns this recommendation to ops; this appendix records the proposed rule.

### D4. Data location

- **Options:** a directory beside the other apps now or an operator-created share first; recommend the directory now and a share with a quota later.
- **Decided:** 2026-11-24 design recommendation for ops to adopt as a reversible call.
- **Rule:** use a directory beside the other apps on the storage host's volume. Ops does not create DiskStation Manager (DSM) shares; the operator creates the later share and quota.
- **Undo:** stop Zot, copy the data to an operator-approved location, update the mount, and prove pulls before restart for consumers.
- **Record:** the 2026-11-24 registry brief assigns this recommendation to ops; this appendix records the proposed rule.

### D5. Agent pull credentials

- **Options:** one read-only Zot credential per agent account or direct image loads; recommend the credential per account under the existing ruling.
- **Decided:** 2026-11-24, by the coordination role as a reversible call after the operator ordered the resume.
- **Rule:** each agent account's rootless daemon pulls with its own read-only Zot credential from Connect, the 1Password Connect service. Keep it separate from every push credential.
- **Undo:** revoke it and return to direct image loads.
- **Record:** the 2026-11-24 Decisions entry and answers, as supplied in the registry brief; original record unverified in this run.

### D6. Docker Hub account

- **Options:** anonymous on-demand Docker Hub sync now or an account now; recommend anonymous sync under the existing ruling.
- **Decided:** 2026-11-24, by the coordination role as a reversible call after the operator ordered the resume.
- **Rule:** use anonymous Docker Hub on-demand sync and caching, with no Docker Hub account for now. An account becomes an operator ask only if rate limits bite.
- **Undo:** create an account and add its credential to sync after the operator answers that ask.
- **Record:** the 2026-11-24 Decisions entry and answers, as supplied in the registry brief; original record unverified in this run.

### D7. Runtime and reachability

- **Options:** Zot under `runc` on the storage host with tailnet-only access or stop the project; recommend proceeding under the existing ruling.
- **Decided:** 2026-11-14, by the operator, Open 228; the operator ordered the resume on 2026-11-24.
- **Rule:** Zot runs on the storage host under `runc` and remains reachable over the tailnet only. The loopback spike precedes tailnet exposure.
- **Undo:** stop the project.
- **Record:** Open 228 and the 2026-11-24 resume, as supplied in the registry brief; original records unverified in this run.
