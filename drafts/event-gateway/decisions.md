<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->

# Event gateway decisions

Back to [the design](README.md).

## Open

### D1. May the gateway use the Go module path `go.tbhb.dev/event-gateway`?

- **Recommendation:** yes.
- **Why:** it matches `go.tbhb.dev/awrel`, and sessions run the client with `go run`, so the path lands in every harness config.
- **If yes:** step 0 adds the vanity entry.
- **If no:** name another path before step 1.
- **Reversible:** costly after release, since published versions keep the path.

### D2. Should the session endpoint stay off public ingress?

- **Recommendation:** yes. The tunnel routes only the webhook paths, and sessions and the read API connect over the private network.
- **Why:** every session runs on an owned host, and awrel's public endpoint with one shared token is its largest exposure. github-metrics made the same call for its tunnel.
- **If yes:** a session off the private network, such as a hosted cloud session, can't connect.
- **If no:** add `/connect` and the read API to the tunnel route. Per-client tokens still apply.
- **Reversible:** yes, by changing the tunnel route.

### D3. May the gateway keep raw event payloads at rest for 7 days?

- **Recommendation:** yes, on the owned host's volume only, never in logs or traces, with a sweep that deletes them after 7 days.
- **Why:** replay and the read API need payloads. Payloads from private repositories carry private content.
- **If yes:** replay covers a week. Backups of the volume, if any, follow the same window.
- **If no:** store envelopes only. Replay then sends envelopes, and sessions fetch details from GitHub.
- **Reversible:** yes, going forward. Data already copied stays copied.

### D4. May the gateway hold a GitHub credential to request redelivery of failed webhooks?

- **Recommendation:** yes, a dedicated GitHub App with only the webhook permission it needs, installed on the organization, in step 9.
- **Why:** GitHub doesn't retry a failed delivery on its own. Without redelivery, an outage of the one host loses events, as awrel's disconnects do.
- **If yes:** ops stores the App key in 1Password and delivers it as a file.
- **If no:** an operator redelivers by hand from GitHub's delivery log after an outage.
- **Reversible:** yes, by uninstalling the App.

### D5. Should the gateway's GitHub webhook be organization-wide?

- **Recommendation:** yes, one organization webhook for the events sessions subscribe to today, added beside awrel's.
- **Why:** one webhook and one secret cover every repository, including new ones, and step 10 removes one webhook rather than many.
- **If yes:** an organization owner creates it, and every repository's events reach the owned host.
- **If no:** per-repository webhooks, added and removed per repository.
- **Reversible:** yes.

### D6. May tbhb-dev publish the gateway's container image as a public package?

- **Recommendation:** yes, to GitHub's container registry, pushed only from `main` with the workflow's own token, and deployed by digest.
- **Why:** the same question is open for agent-capacity. One answer should cover both.
- **If yes:** step 2 pulls by digest.
- **If no:** ops builds the image on the host from a pinned commit.
- **Reversible:** yes, by deleting package versions.

## Decided

The operator decided these before the design:

- **The gateway is a new project.** tbhb-dev/event-gateway, public, self-hosted on owned hosts, and the front door for event-driven work. It supersedes awrel, whose code stays unchanged until it retires.
- **No paid services.** Public ingress uses a Cloudflare Tunnel.
- **Telemetry.** Every service sends OpenTelemetry to the loopback-only local collector, and logs go to Loki.
- **End state.** Temporal on owned hosts runs workflows behind the gateway, workflows launch agents through `tbhbdev alpha agent exec`, and intent sessions replace coordination sessions.
