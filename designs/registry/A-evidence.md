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

Labels distinguish evidence from intent. **Proven** means this run observed a result, **existing** means a source or supplied authority describes something already present, **new** means proposed work, and **unverified** means this run did not establish the claim. A vendor claim proves neither local behavior nor compatibility with the pinned binary.

## Authority and existing constraints

- **Existing, supplied authority:** the 2026-11-24 Decisions entry and operator answers take precedence over the tracker, then the 2026-11-14 research report. This run read the brief, not those original records, so their independent verification remains **unverified**.
- **Existing, supplied authority:** Open 228, dated 2026-11-14, selects Zot on the storage host under `runc` with tailnet-only access; undo means stop the project. The operator ordered the resume on 2026-11-24; D7 records this authority.
- **Existing, supplied authority:** the coordination role chose per-agent-account read-only credentials in Connect, separate from push credentials, and anonymous Docker Hub sync without an account on 2026-11-24. D5 and D6 record both rules and their undo paths.
- **Existing, supplied authority:** the 2026-11-12 Decisions entry keeps agent-capacity as a direct load until the registry serves it. A later follow-up moves it to registry pulls.
- **Existing, supplied authority:** Open 161 already decided and applied removal of the agent from the storage host's Docker group. That work stays outside this design; this run did not inspect host membership.
- **Unverified:** storage-host capacity, filesystem support, deployed Docker versions, network policy, and outage behavior. The brief names the storage host's Docker 24 as a required client, not as a version this run checked.
- **New:** one storage-host registry remains a single point of failure with neighboring services. Running containers keep running during a registry outage, while new deployments use direct image loads; step 3 must check that fallback before consumers depend on it.

## Compatibility and signatures

- **Existing, vendor claim:** the supplied 2026-11-14 report digest says Zot uses Apache-2.0 and rejects Docker schema 2 unless `http.compat` contains `docker2s2`. The [versioned installation guide](https://zotregistry.dev/v2.1.23/install-guides/install-guide-linux/) and [mirroring guide](https://zotregistry.dev/v2.1.23/articles/mirroring/), read 2026-11-24, provide the deployment and compatibility context; this run did not independently read the license or exercise rejection.
- **New:** build-time digest pinning of Zot v2.1.23, Renovate updates, and `http.compat` with `docker2s2`.
- **Unverified:** classic-builder Docker schema 2 push, buildx Open Container Initiative (OCI) push, and pulls from the lab hosts' Docker and the storage host's Docker 24. Rollout check 1 must prove all four paths.
- **Existing, vendor claim:** the [versioned signature guide](https://zotregistry.dev/v2.1.23/articles/verifying-signatures/), read 2026-11-24, describes signature verification. It does not prove that the proposed clients can sign and attest through the deployed referrers API.
- **Unverified:** `cosign sign`, `cosign attest`, and `cosign verify` through the referrers API. Check 2 records those results with the pinned build before later enforcement work.
- **Unverified, report claim:** Distribution had no merged referrers API as of 2026-11-14. This run did not inspect its source or change history.
- **Unverified, report claim:** Harbor needs PostgreSQL, Valkey, and about 4 GB of memory. This run did not read Harbor documentation or measure a deployment; the memory figure is a reported software requirement, not a host capacity or quota.

## Storage and cleanup

- **Existing, vendor claim:** the [Zot v2.1.23 storage guide](https://zotregistry.dev/v2.1.23/articles/storage/), read 2026-11-24, describes online garbage collection (GC), `gcDelay` with a one-hour default, and hard-link-based deduplication. Host support and behavior under load remain **unverified**.
- **Proven, documentation read:** the same storage page says omitted `gcInterval` means no periodic collection in its attribute table, but its GC configuration table says the omitted value gives a one-hour interval. This conflict requires an explicit interval and a pinned-build check; it does not establish the runtime default.
- **Existing, vendor claim:** the storage guide says Zot v2.1.21 made blob reads repository-local by default. A digest in the global deduplication cache alone does not establish that a repository can serve it; check cross-repository pushes, pulls, and explicit blob mounts in step 3.
- **Existing, vendor claim:** the [retention guide](https://zotregistry.dev/v2.1.23/articles/retention/), read 2026-11-24, describes tag-pattern, count, and age retention with `dryRun`. Once a repository has a `keepTags` policy, tags matching no keep rule face deletion; its recommended catch-all policy comes last.
- **Existing, vendor claim:** the retention guide says `verify-feature retention` needs a stopped local server to avoid database locks, and its GC can delete orphan blobs even when retention uses dry-run mode. Run destructive verification only on disposable spike data; retention dry runs do not make GC a dry run.
- **Existing, vendor claim:** the [Distribution GC guide](https://distribution.github.io/distribution/about/garbage-collection/), read 2026-11-24, requires read-only mode or a stopped registry during its stop-the-world GC. Concurrent writes can corrupt images.
- **New:** local file storage, explicit periodic GC configuration, default GC delay, retention dry runs for a week, and a catch-all keep policy last. Real deletion follows the logged candidate comparison in check 4.
- **Unverified:** check 4's predicted versus actual deletions, resident memory at idle and during parallel multi-GB pushes, usable capacity, hard-link behavior, and a suitable memory limit. Check 5 measures memory before ops chooses that limit.

## Authentication and authorization

- **Existing, vendor claim:** the [Zot v2.1.23 authentication and authorization guide](https://zotregistry.dev/v2.1.23/articles/authn-authz/), read 2026-11-24, lists htpasswd, Lightweight Directory Access Protocol (LDAP), bearer tokens, and OpenID Connect (OIDC) workload identity. It also lists mutual Transport Layer Security and API keys; these are supported alternatives, not proposed credentials for this design.
- **Existing, vendor claim:** that guide supports only bcrypt for htpasswd and describes per-repository `read`, `create`, `update`, and `delete` grants with longest-glob matching. Its instructions require `read` alongside write actions, which explains the proposed publisher grants.
- **Existing, vendor claim:** the guide provides authenticated catalog access controls and separate metrics access policy. No anonymous repository policy plus anonymous catalog off still needs denial tests on the pinned service.
- **New:** a Python renderer in the tbhbops command-line tool reads 1Password references through Connect and writes protected htpasswd output, following the object store identity pattern. Public configuration uses `<secret-reference>` placeholders; only private configuration contains actual 1Password URI references.
- **New:** publishers get per-repository read/create/update from main-branch continuous integration (CI) jobs only; production daemons and each agent account get distinct read-only credentials. A metrics-only identity serves Alloy, and the global administrator credential stays break-glass only.
- **Unverified:** current object store renderer details, production Docker config delivery, rootless daemon login behavior, credential rotation, cross-repository denial, and anonymous denial. Steps 2, 3, and 7 test these paths without logging credentials.

## Mirrors and upstream limits

- **Existing, vendor claim:** the [Zot v2.1.23 mirroring guide](https://zotregistry.dev/v2.1.23/articles/mirroring/), read 2026-11-24, describes on-demand and periodic sync with per-prefix destinations. It warns against Docker Hub catalog polling and supports retaining upstream path segments beneath a destination prefix.
- **Existing, vendor claim:** the [Distribution mirror recipe](https://distribution.github.io/distribution/recipes/mirror/), read 2026-11-24, describes Docker Hub pull-through caching. The Docker daemon mirror setting covers Docker Hub only; clients need explicit references for the other proposed upstream prefixes.
- **Existing, vendor claim:** [Docker Hub usage](https://docs.docker.com/docker-hub/usage/), read 2026-11-24, lists 100 anonymous pulls per IPv4 address per six hours. A shared egress uses that allowance; this run did not measure local pull demand.
- **Unverified, report claim:** the supplied digest describes an earlier sync credential-forwarding failure. This run did not inspect the failing build, patch, or security report; fixed anonymous upstreams are a constraint, not proof that the failure is absent.
- **New:** anonymous on-demand sync from `docker.io`, `ghcr.io`, and `quay.io` into `mirror/<upstream>/**`, with full upstream paths and identical digests. No Docker Hub account now, and no upstream credential forwarding.
- **Unverified:** check 3's digest comparison and credential-boundary test for all three upstreams, with request evidence that excludes secret values. Step 4 uses temporary sync configuration; step 8 repeats the check before routine use.

## Deployment and operational patterns

- **Existing, supplied pattern:** the brief names the object store project's `apps/s3` hardening and identity rendering, plus the observability stack's Alloy wiring. Their implementations remain **unverified** because this run did not read them; the design proposes following the pattern without claiming it already matches.
- **New:** non-root containers, `cap_drop: ALL`, `no-new-privileges`, `pids_limit`, bounded `json-file` logs, and a memory limit from the spike. Step 2 defines the configuration, and step 3 tests it.
- **New:** Alloy scrapes Zot Prometheus metrics into the metrics store and forwards JSON logs to the log store. Dashboards cover pushes, pulls, latency, and disk; alerts cover downtime, 5xx rate, GC errors, and storage use.
- **Unverified:** collector placement, metric names, scrape authentication, current log wiring, and delivery to the observability host. Step 6 checks dashboards and triggers each alert.
- **New:** a loopback-only first start, then a tailnet-only Tailscale Serve sidecar after D1. Check 5 measures multi-GB push and pull throughput through the sidecar; this run measured neither throughput nor memory.
- **New:** bootstrap components keep direct upstream images, and deployment uses merged main only. Repository names become a one-way door once digest-pinned Compose references depend on them.
- **Unverified:** current network policy, sidecar Service support, and accept and deny behavior. Step 4 checks these after the operator records D1; this document does not authorize a policy merge.

## Backup and restore

- **New:** a local file-level copy taken while Zot runs, followed by an isolated restore that serves pulls, is acceptance check 6. The copy includes content, metadata, and recovery configuration; secrets stay within protected recovery procedures.
- **Unverified:** live-copy consistency, filesystem copy semantics, secondary NAS access and capacity, and recovery of the sidecar and credential renderer. A vendor's online GC claim does not prove backup consistency.
- **New:** D2 gates a backup job to the secondary NAS and a second restore drill in step 5. If D2 stays open, no job touches that destination.

## Design sources and scope of this run

- **Proven, local read:** this run read `designs/WRITING.md`, `designs/REVIEW.md`, and `designs/TEMPLATE.md` from `origin/main`, plus the event-gateway README, evidence, and decisions model from that ref on 2026-11-24. This design follows their frontmatter and appendix style; the supplied registry skeleton sets its section order and three-file limit.
- **Unverified:** the graph-postgres model from tbhb-dev/wiki#152. Fetching `agent/gpgdesign3` failed because the remote lacked that branch, and the pull request head ref was also unavailable; this run could not read that model.
- **New:** the build and deployment checks belong to later runs. This documentation run proves no registry deployment, runtime capacity, policy change, backup, credential, or acceptance check.
