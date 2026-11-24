---
type: Design Draft
title: Internal container registry
description: A private container registry and upstream cache for image publishers and consumers.
tags: [registry, containers, design]
status: draft
owner: Registry maintainer
updated: "2026-11-24"
decisions_open: 2
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Internal container registry

| Status | Owner | Updated | Needs from the operator |
| --- | --- | --- | --- |
| Draft | Registry maintainer | 2026-11-24 | 2 open items in [decisions.md](decisions.md) |

## Summary

Recommend Zot as the internal container registry and cache for public upstream images. Publishers push once, and consumers pull digest-pinned images over the tailnet. Separate credentials limit each publisher and consumer to its repositories.

A private companion design holds the host-level details.

The proposal runs Zot on the storage host, with a Tailscale Serve sidecar and file storage. A staged rollout checks Docker compatibility, signatures, mirrors, cleanup, resource use, and restore before clients depend on it. The operator has two open asks: network policy and backup to the secondary NAS.

## Proposal

This is a new service, not a deployed one. [Appendix A](A-evidence.md) labels the evidence and the gaps behind each part.

| Component | Runs where and keeps what | Responsible role |
| --- | --- | --- |
| Zot v2.1.23 | The storage host under `runc`; image blobs and metadata in a directory beside the other apps | Registry maintainer, with ops deploying merged main |
| Tailscale Serve sidecar | The storage host; persistent sidecar identity state beside the app | Ops maintains the sidecar and network policy |
| Credential renderer | The tbhbops command-line tool; bcrypt output beside the app, with source secrets in 1Password Connect | Ops maintains the renderer and identities |
| Image publishers | The CI runner pool; per-repository push credentials in the job's secret delivery | Build maintainer limits jobs to main |
| Image consumers | Production daemons and each agent account's rootless daemon; local Docker credential config | Ops manages production; an agent account manages its own daemon |
| Telemetry collector | Alloy forwards registry telemetry to the observability host's metrics and log stores | Observability maintainer owns wiring, dashboards, and alerts |
| Backup job | Copies registry files to the secondary NAS after D2; backup state stays there | Backup operator owns scheduling and restore drills |

Choices use one recommendation each. D1 through D7 live in [decisions.md](decisions.md); the remaining reversible implementation choices follow here.

| Choice | Options and recommendation |
| --- | --- |
| Registry software | Zot, Distribution, or Harbor; recommend Zot for one service with online cleanup and upstream sync |
| Authentication | bcrypt htpasswd, directory authentication, or token federation; recommend bcrypt htpasswd through Connect |
| Transport | Tailscale Serve or a separate reverse proxy; recommend Tailscale Serve with Transport Layer Security |
| Bootstrap source | Direct upstream images or registry images; recommend direct upstream images for bootstrap components |
| Mirror routing | Explicit image paths or daemon mirror settings; recommend explicit paths |
| Telemetry | Existing Alloy pipelines or a separate stack; recommend Alloy |
| Image updates | Digest pins with Renovate or floating tags; recommend digest pins with Renovate |
| Cleanup | Staged dry runs or immediate deletion; recommend staged dry runs with an explicit periodic interval |
| Hardening | Object store app controls or runtime defaults; recommend the object store app controls |
| Repository naming | Stable names before migration or names that change per deployment; recommend stable names before migration |

## How it works

### Storage

New: Zot stores blobs and metadata in a directory beside the other apps on the storage host's volume ([D4](decisions.md#d4-data-location)). Ops sets ownership for the non-root container and keeps the data outside its writable layer. An operator-created share with a quota comes later; ops does not create shares in DiskStation Manager (DSM).

Unverified: not measured yet: host capacity and filesystem hard-link support. The local spike measures memory and disk behavior before ops sets limits or enables filesystem-dependent optimizations ([storage evidence](A-evidence.md#storage-and-cleanup)).

### Network

The first start binds only to loopback, and checks use a Secure Shell (SSH) tunnel. The later sidecar serves a Tailscale Service over the tailnet, using a name under the internal domain. Zot has no public listener.

New: the sidecar needs a tag for the registry sidecar and grants for approved clients. D1 needs the operator's recorded answer before any policy pull request merges. Policy tests must accept approved publishers and consumers and deny unrelated clients ([network evidence](A-evidence.md#deployment-and-operational-patterns)).

### Clients

The continuous integration (CI) runner pool pushes only from main-branch jobs with a separate push identity per repository. Production daemons pull with their own read-only credential in root's Docker config. Each agent account's rootless daemon pulls with its own read-only credential from Connect, separate from every push credential ([D3](decisions.md#d3-push-identities), [D5](decisions.md#d5-agent-pull-credentials)).

Bootstrap images never come from the registry. This includes the registry's own image, the Tailscale sidecar, 1Password Connect, the object store, and the DNS server. The agent-capacity image stays a direct load until the registry serves it; migration to a registry pull is a later follow-up ([authority](A-evidence.md#authority-and-existing-constraints)).

Clients write `mirror/docker.io/**`, `mirror/ghcr.io/**`, or `mirror/quay.io/**` beneath the registry name in their image references. The Docker daemon mirror setting covers Docker Hub only. Consumers pin digests, and the namespace preserves upstream repository paths, including Docker Hub's library prefix ([mirror evidence](A-evidence.md#mirrors-and-upstream-limits)).

### Auth

New: ops renders htpasswd with bcrypt from 1Password through Connect, following the object store identities pattern. Public configuration contains only placeholder secret references such as `<secret-reference>`; private configuration holds the 1Password URI references, never secret values. A Python subcommand in the tbhbops command-line tool writes the protected output atomically and keeps secrets out of logs.

Push identities get per-repository `read`, `create`, and `update`, with no `delete`. Pull identities get `read` only. Zot's vendor documentation says the longest matching glob wins, so checks cover overlapping rules and cross-repository denials ([auth evidence](A-evidence.md#authentication-and-authorization)).

Omit anonymous policies and turn anonymous catalog access off. Keep the global administrator identity for break-glass use only, outside routine jobs. Give Alloy a metrics-only identity, and check that unauthenticated pulls, pushes, catalog listing, and metric reads fail.

### Observability

New: Alloy scrapes Zot's Prometheus metrics endpoint into the metrics store and forwards JSON container logs into the log store. Follow the object store app's pattern and the observability stack's Alloy wiring. Unverified: no read of the deployment configuration, collector placement, endpoint authentication, or current wiring yet.

Dashboards show pushes, pulls, request latency, and disk use. Alerts cover registry down, elevated 5xx rate, garbage collection (GC) errors, and storage use. Step 6 verifies metric names and labels against the pinned build and tests log delivery before clients rely on alerts ([operational evidence](A-evidence.md#deployment-and-operational-patterns)).

### Lifecycle and mirrors

New: pin Zot v2.1.23 by digest at build time, and let Renovate propose updates. Add `docker2s2` to `http.compat` for Docker schema 2, alongside Open Container Initiative (OCI) image support. The classic builder, buildx, and consumer checks remain unverified until the spike ([compatibility evidence](A-evidence.md#compatibility-and-signatures)).

Turn GC on with the default `gcDelay` of one hour. Set `gcInterval` explicitly after the spike, because the storage documentation disagrees with itself about the omitted interval. Start retention in `dryRun`, with tag-pattern, count, and age rules and a catch-all `keepTags` policy last. Zot's vendor documentation warns that once any keep rule applies, it removes every tag that matches no rule ([cleanup evidence](A-evidence.md#storage-and-cleanup)).

New: use on-demand anonymous sync from fixed `docker.io`, `ghcr.io`, and `quay.io` upstreams into separate `mirror/<upstream>/**` prefixes. Keep the complete upstream paths and digests. No Docker Hub account exists for this project now; ask again only if rate limits bite ([D6](decisions.md#d6-docker-hub-account)).

The credential-forwarding check must show that client credentials stay at the mirror. Keep upstream endpoints fixed and anonymous because an earlier research report (internal record) describes an earlier sync credential-forwarding failure. Vendor support for periodic sync does not change the on-demand recommendation, and Docker Hub must not use catalog polling ([mirror evidence](A-evidence.md#mirrors-and-upstream-limits)).

Apply the object store app's hardening pattern: non-root, `cap_drop: ALL`, `no-new-privileges`, and `pids_limit`. Set bounded `json-file` logs and a memory limit that the spike sizes. The current object store configuration and suitable limits remain unverified.

### Backup and restore

New: a file-level copy includes registry content and metadata, app configuration, and the sidecar's identity state. Secret material stays within the protected backup and 1Password recovery procedures, outside public files. A local copy taken while Zot runs must restore into an isolated registry that serves digest-pinned pulls before anyone treats the method as safe.

The backup operator adds a job to the secondary NAS only after D2. A later restore drill checks the copied data, configuration, secret rendering, and sidecar recovery without reusing a live service identity. Copy consistency and the backup destination's current setup remain unverified ([restore evidence](A-evidence.md#backup-and-restore)).

## Rollout

These are planned build and deploy runs, not completed checks. Deploy only from merged main; each step records evidence and links it from the tracker.

1. **Merge the design pull requests.** Owner: registry maintainer. Exposure: public design text only. Done when both design reviews clear their exact heads, the paired designs agree, and the operator answers D1 and D2 before merge under the repository review rules. Undo: supersede the design or stop the project.
2. **Merge the app implementation.** Owner: ops. Exposure: repository changes only; this step starts nothing. Add the registry app with Compose, Zot configuration, and the htpasswd renderer as a Python subcommand in the tbhbops command-line tool, with renderer and policy tests. Done when tests cover secret handling, bcrypt output, least-privilege grants, no anonymous catalog, and digest pins on merged main. Undo: revert the app changes.
3. **Start on loopback.** Owner: ops. Exposure: the storage host and an SSH tunnel only. Create the data directory and start from merged main. Done when **check 1** pushes Docker schema 2 from the classic builder and OCI from buildx, then pulls both with the lab hosts' Docker and the storage host's Docker 24; **check 4** compares GC and retention dry-run logs with documented results, then proves real GC deletes only the expected disposable data; **check 5, memory** records resident memory at idle and during a parallel multi-GB push to size the limit; and **check 6, local copy** restores a file-level copy taken while Zot runs into an isolated registry that serves pulls. Undo: stop Zot, preserve the data, and keep direct image loads.
4. **Add the tailnet route.** Owner: network operator and ops. Exposure: approved tailnet clients only. Merge the policy pull request only after D1's recorded answer and accept and deny tests, then deploy the sidecar to serve the Tailscale Service from merged main. Done when **check 5, throughput** records a multi-GB push and pull through Tailscale Serve; **check 2** proves `cosign sign`, `cosign attest`, and `cosign verify` through the referrers API; and **check 3** proves on-demand sync from all three upstreams returns identical digests while client credentials stay at the mirror. Use a temporary spike configuration for sync, then leave routine mirrors off until step 8. Undo: stop the sidecar and revert the route and grants.
5. **Add the backup job and restore drill.** Owner: backup operator. Exposure: the secondary NAS, only after D2. Done when the job from merged main copies the live registry and a fresh restore serves digest-pinned pulls, repeating check 6 at the backup destination. Undo: stop the job and remove its grants, keeping the source data.
6. **Connect telemetry.** Owner: observability maintainer. Exposure: authenticated collector access and the existing metrics and log stores. Done when Alloy delivers metrics and JSON logs, dashboards show all four views, and test failures trigger each named alert. Undo: revert collector wiring, dashboards, and alerts.
7. **Enable publishers.** Owner: build maintainer and ops. Exposure: per-repository write access for main-branch CI jobs only. Done when a main job pushes, another repository and a non-main job fail to push, production credentials only pull, and each agent account only pulls with its own Connect credential. Undo: revoke the new identities and return affected clients to direct loads.
8. **Enable mirrors.** Owner: ops. Exposure: outbound requests to the three fixed public upstreams. Done when check 3 passes again with routine on-demand configuration from merged main, explicit client paths work, and bootstrap references still use upstream sources. Undo: turn sync off and restore direct upstream references.
9. **Enable retention deletion.** Owner: registry maintainer and ops. Exposure: deletion of eligible registry data. Done when a week of logged dry runs matches the expected candidate list, the catch-all keep rule protects unmatched tags, and check 4 passes again before `dryRun` turns off through merged main. Undo: restore dry-run configuration and recover deleted content from backup or its publisher.

## Risks

| Risk | Effect | Mitigation or first check | Evidence |
| --- | --- | --- | --- |
| One storage host holds the registry and other services | A host failure blocks fresh pulls and deployments | Running containers keep running; new deploys fall back to direct image loads | Existing constraint; outage behavior unverified locally, [A](A-evidence.md#authority-and-existing-constraints) |
| Host capacity and hard-link support are unknown | Writes fail or resource use harms neighboring services | Step 3 measures capacity, resident memory, and filesystem behavior before limits | Unverified: not measured yet, [A](A-evidence.md#storage-and-cleanup) |
| `keepTags` defaults to deleting unmatched tags once rules apply | A rule mistake removes needed tags | Catch-all keep policy last, dry runs for a week, and a tested restore | Vendor claim, [A](A-evidence.md#storage-and-cleanup) |
| `gcInterval` documentation conflicts | Cleanup runs at an unexpected frequency | Set the interval explicitly and observe step 3 | Existing documentation conflict, [A](A-evidence.md#storage-and-cleanup) |
| Zot v2.1.21 changed blob reads to repository-local | A shared digest fails in a repository without its local blob link | Step 3 tests cross-repository pushes, pulls, and explicit blob mounts | Vendor claim, [A](A-evidence.md#storage-and-cleanup) |
| Earlier sync credential-forwarding failure | A mirror request exposes a client credential upstream | Fixed anonymous upstreams and check 3 before mirror use | Existing (internal record): an earlier research report; unverified behavior, [A](A-evidence.md#mirrors-and-upstream-limits) |
| Repository names enter every digest-pinned Compose file | Renaming breaks consumer references | Treat names under the registry service as a one-way door and settle them before migration | New constraint, [A](A-evidence.md#deployment-and-operational-patterns) |
| Anonymous upstream limits or outages | A cache miss fails during deployment | Preserve bootstrap sources and direct loads; revisit an account only after rate-limit evidence | Vendor limit claim, [A](A-evidence.md#mirrors-and-upstream-limits) |
| A live file copy misses consistent metadata | A restore fails to serve images | Check 6 gates dependence on the copy method | Unverified: not measured yet, [A](A-evidence.md#backup-and-restore) |

## Later

These are deferred options, not requirements for this rollout.

A general internal package registry could later serve Python packages, container images, and Go modules outside the current build. Unverified: confirm against the vendor documentation when the work starts: Zot supports OCI only, storing OCI images and OCI artifacts without speaking the Python package index or Go module proxy protocols. Clients reach Zot through one stable service name behind the tailnet proxy with the same per-repository authentication. Another package server could sit beside Zot behind the same proxy and authentication, or a multi-format server could replace Zot without changing consumer references, provided repository names remain one-way doors as the Risks section says. Nothing in the current design blocks this later work.

| Follow-up | Options and recommendation |
| --- | --- |
| Agent-capacity image | Keep direct loads or use registry pulls after the registry serves it; recommend the later pull migration |
| Metrics-exporter image | Direct upstream pulls or a mirror; recommend adding a mirror later |
| Signature enforcement | Advisory verification or enforcement in the deploy account's receive program; recommend later enforcement |
| Runtime placement | Keep `runc` on the storage host or move to `runsc` elsewhere; recommend a later placement study off the storage host |
| Storage backend | Local files or an S3-compatible object store backend; recommend evaluating the backend later |

Evidence: [A. Evidence](A-evidence.md). Open items and authority: [decisions.md](decisions.md).
