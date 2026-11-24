---
type: Design Appendix
title: Internal container registry evidence
description: Source claims, existing authority, proposed work, and checks that remain unverified.
tags: [registry, evidence, design]
status: draft
parent: README.md
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Appendix A. Evidence

Back to [the design](README.md).

Labels distinguish evidence from intent. **Proven** means a recorded observation, **existing** means a source describes something already present, **new** means proposed work, and **unverified** means a claim still needs a check. A vendor claim proves neither local behavior nor compatibility with the pinned binary.

## Authority and existing constraints

- **Existing (internal record):** the 2026-11-24 decisions entry and operator answers take precedence over the tracker, then the 2026-11-14 research report. **Unverified: no read of those original records yet.**
- **Existing (operator answer relayed by the coordination role):** the 2026-11-24 Open 290 decisions entry approves D1 and D2. D1 adds a registry tag, a Tailscale Service, and least-privilege grants; both the internal DNS name and the Tailscale Service are `images.tbhb.cloud`. D2 approves secondary NAS backup as follow-up infrastructure debt, outside the critical path to standing the registry up.
- **Existing (internal record):** the Open 228 decision, dated 2026-11-14, selects Zot on the storage host under `runc` with tailnet-only access; undo means stop the project. The operator ordered the resume on 2026-11-24; D7 records this authority.
- **Existing (internal record):** the 2026-11-24 decisions entry records that the coordination role chose per-agent-account read-only credentials in Connect, separate from push credentials, and anonymous Docker Hub sync without an account on 2026-11-24. D5 and D6 record both rules and their undo paths.
- **Existing (internal record):** the 2026-11-12 decisions entry keeps agent-capacity as a direct load until the registry serves it. A later follow-up moves it to registry pulls.
- **Existing (internal record):** the Open 161 decision records removal of the agent from the storage host's Docker group as decided and applied. That work stays outside this design. **Unverified: no read of host membership yet.**
- **Unverified: not measured yet:** storage-host capacity, filesystem support, deployed Docker versions, network policy, and outage behavior. The storage host's Docker 24 is a required client; its deployed version remains unverified.
- **New:** one storage-host registry remains a single point of failure with neighboring services. Running containers keep running during a registry outage, while new deployments use direct image loads; step 3 must check that fallback before consumers depend on it.

## Compatibility and signatures

- **Existing (internal record):** an earlier research report, dated 2026-11-14, says Zot uses Apache-2.0 and rejects Docker schema 2 unless `http.compat` contains `docker2s2`. The [versioned installation guide](https://zotregistry.dev/v2.1.23/install-guides/install-guide-linux/) and [mirroring guide](https://zotregistry.dev/v2.1.23/articles/mirroring/), read 2026-11-24, provide the deployment and compatibility context. **Unverified: no read of the license yet; not measured yet:** schema rejection.
- **New:** build-time digest pinning of Zot v2.1.23, Renovate updates, and `http.compat` with `docker2s2`.
- **Unverified: not measured yet:** classic-builder Docker schema 2 push, buildx Open Container Initiative (OCI) push, and pulls from the lab hosts' Docker and the storage host's Docker 24. Rollout check 1 must prove all four paths.
- **Existing, vendor claim:** the [versioned signature guide](https://zotregistry.dev/v2.1.23/articles/verifying-signatures/), read 2026-11-24, describes signature verification. It does not prove that the proposed clients can sign and attest through the deployed referrers API.
- **Unverified: not measured yet:** `cosign sign`, `cosign attest`, and `cosign verify` through the referrers API. Check 2 records those results with the pinned build before later enforcement work.
- **Existing (internal record):** an earlier research report says Distribution had no merged referrers API as of 2026-11-14. **Unverified: no read of its source or change history yet.**
- **Existing (internal record):** an earlier research report says Harbor needs PostgreSQL, Valkey, and additional memory for those services. **Unverified: no read of Harbor documentation yet; not measured yet:** deployment memory use.

## Storage and cleanup

- **Existing, vendor claim:** the [Zot v2.1.23 storage guide](https://zotregistry.dev/v2.1.23/articles/storage/), read 2026-11-24, describes online garbage collection (GC), `gcDelay` with a one-hour default, and hard-link-based deduplication. Host support and behavior under load remain **unverified**.
- **Proven, documentation read:** the same storage page says omitted `gcInterval` means no periodic collection in its attribute table, but its GC configuration table says the omitted value gives a one-hour interval. This conflict requires an explicit interval and a pinned-build check; it does not establish the runtime default.
- **Existing, vendor claim:** the storage guide says Zot v2.1.21 made blob reads repository-local by default. A digest in the global deduplication cache alone does not establish that a repository can serve it; check cross-repository pushes, pulls, and explicit blob mounts in step 3.
- **Existing, vendor claim:** the [retention guide](https://zotregistry.dev/v2.1.23/articles/retention/), read 2026-11-24, describes tag-pattern, count, and age retention with `dryRun`. Zot selects the first matching repository policy; once that policy has a `keepTags` rule, Zot deletes tags that match no keep rule. A final catch-all policy protects only repositories that no earlier policy matches; pinned-build behavior remains **unverified**.
- **Existing, vendor claim:** the retention guide says `verify-feature retention` needs a stopped local server to avoid database locks, and its garbage collection deletes orphan blobs even when retention uses dry-run mode. Run this verifier only on disposable spike data; retention previews do not preview garbage collection. **Unverified: not measured yet:** the pinned build's preview output and actual orphan-blob deletion.
- **Existing, vendor claim:** the [Distribution GC guide](https://distribution.github.io/distribution/about/garbage-collection/), read 2026-11-24, requires read-only mode or a stopped registry during its stop-the-world GC. Concurrent writes can corrupt images.
- **New:** local file storage, explicit periodic garbage collection configuration, default collection delay, retention previews for a week, and explicit keep rules for protected tags in each repository policy. A final catch-all policy protects only repositories that no earlier policy matches; check 4 compares retention candidates and tests real garbage collection only on disposable data before ops enables retention deletion.
- **Unverified: not measured yet:** check 4's predicted versus actual deletions, resident memory at idle and during parallel pushes, usable capacity, hard-link behavior, and a suitable memory limit. Check 5 measures memory before ops chooses that limit.

## Authentication and authorization

- **Existing, vendor claim:** the [Zot v2.1.23 authentication and authorization guide](https://zotregistry.dev/v2.1.23/articles/authn-authz/), read 2026-11-24, lists htpasswd, Lightweight Directory Access Protocol (LDAP), bearer tokens, and OpenID Connect (OIDC) workload identity. It also lists mutual Transport Layer Security and API keys; these are supported alternatives, not proposed credentials for this design.
- **Existing, vendor claim:** that guide supports only bcrypt for htpasswd and describes per-repository `read`, `create`, `update`, and `delete` grants with longest-glob matching. Its instructions require `read` alongside write actions, which explains the proposed publisher grants.
- **Existing, vendor claim:** the guide provides authenticated catalog access controls and separate metrics access policy. No anonymous repository policy plus anonymous catalog off still needs denial tests on the pinned service.
- **New:** a Python renderer in the tbhbops command-line tool reads 1Password references through Connect and writes protected htpasswd output, following the object store identity pattern. Public configuration uses `<secret-reference>` placeholders; only private configuration contains actual 1Password URI references.
- **New:** publishers get per-repository read/create/update from main-branch continuous integration (CI) jobs only; production daemons and each agent account get distinct read-only credentials. A metrics-only identity serves Alloy, and the global administrator credential stays break-glass only.
- **Unverified: not measured yet:** current object store renderer details, production Docker config delivery, rootless daemon login behavior, credential rotation, cross-repository denial, and anonymous denial. Steps 2, 3, and 6 test these paths without logging credentials.

## Mirrors and upstream limits

- **Existing, vendor claim:** the [Zot v2.1.23 mirroring guide](https://zotregistry.dev/v2.1.23/articles/mirroring/), read 2026-11-24, describes on-demand and periodic sync with per-prefix destinations. It warns against Docker Hub catalog polling and supports retaining upstream path segments beneath a destination prefix.
- **Existing, vendor claim:** the [Distribution mirror recipe](https://distribution.github.io/distribution/recipes/mirror/), read 2026-11-24, describes Docker Hub pull-through caching. The Docker daemon mirror setting covers Docker Hub only; clients need explicit references for the other proposed upstream prefixes.
- **Existing, vendor claim:** [Docker Hub usage](https://docs.docker.com/docker-hub/usage/), read 2026-11-24, lists 100 anonymous pulls per IPv4 address per six hours. A shared egress uses that allowance. **Unverified: not measured yet:** local pull demand.
- **Existing (internal record):** an earlier research report describes an earlier sync credential-forwarding failure. **Unverified: no read of the failing build, patch, or security report yet.** Fixed anonymous upstreams are a constraint, not proof that the failure is absent.
- **New:** anonymous on-demand sync from each fixed upstream registry into `mirror/<upstream>/**`, with full upstream paths and identical digests. No Docker Hub account now, and no upstream credential forwarding.
- **Unverified: not measured yet:** check 3's digest comparison and credential-boundary test for all three upstreams, with request evidence that excludes secret values. Step 4 uses temporary sync configuration; step 7 repeats the check before routine use.

## Deployment and operational patterns

- **Existing (internal record):** an earlier research report names the object store app's hardening and identity rendering, plus the observability stack's Alloy wiring. **Unverified: no read of those implementations yet.** The design proposes following the pattern; a match remains unverified.
- **New:** non-root containers, `cap_drop: ALL`, `no-new-privileges`, `pids_limit`, bounded `json-file` logs, and a memory limit from the spike. Step 2 defines the configuration, and step 3 tests it.
- **New:** Alloy scrapes Zot Prometheus metrics into the metrics store and forwards JSON logs to the log store. Dashboards cover pushes, pulls, latency, and disk; alerts cover downtime, 5xx rate, GC errors, and storage use.
- **Unverified: not measured yet:** collector placement, metric names, scrape authentication, current log wiring, and delivery to the observability host. Step 5 checks dashboards and triggers each alert.
- **New:** a loopback-only first start, then a tailnet-only Tailscale Serve sidecar for `images.tbhb.cloud`, approved in D1. Check 5 measures push and pull throughput through the sidecar. **Unverified: not measured yet:** throughput and memory.
- **New:** bootstrap components keep direct upstream images, and deployment uses merged main only. Repository names become a one-way door once digest-pinned Compose references depend on them.
- **Unverified: not measured yet:** current network policy, sidecar Service support, and accept and deny behavior. With D1 decided, step 4 checks these before deployment.

## Backup and restore

- **New:** a local file-level copy taken while Zot runs, followed by an isolated restore that serves pulls, is acceptance check 6. The copy includes content, metadata, and recovery configuration; secrets stay within protected recovery procedures.
- **Unverified: not measured yet:** live-copy consistency, filesystem copy semantics, secondary NAS access and capacity, and recovery of the sidecar and credential renderer. A vendor's online GC claim does not prove backup consistency.
- **New:** D2 approves a backup job to the secondary NAS and a second restore drill as follow-up infrastructure debt. Neither gates standing the registry up; the local restore check remains in step 3.

## Design scope

- **New:** build and deployment checks remain planned work. **Unverified: not measured yet:** registry deployment, runtime capacity, policy change, backup, credentials, and acceptance checks.
