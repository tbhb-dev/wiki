---
type: Design Appendix
title: Event gateway evidence
description: What reads of awrel, the github-metrics receiver, the launcher, and nats-server show for the event gateway design.
tags: [event-gateway, evidence, design]
status: draft
parent: README.md
generated:
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:2220073242f73359
    at: "2026-11-22T23:11:10Z"
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-opus-5-5
    effort: high
    surface: sdk-cli
    session: sha-256-64:1a832a02df3ab163
    at: "2026-11-23T08:02:35Z"
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-opus-5-5
    effort: high
    surface: sdk-cli
    session: sha-256-64:d55a3d2c551dcd46
    at: "2026-11-23T08:31:19Z"
  - role: generated
    harness: claude-code:2.1.330
    model: anthropic:claude-sonnet-5-5
    effort: medium
    surface: cli
    session: sha-256-64:2220073242f73359
    at: "2026-11-23T08:46:23Z"
---

<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- Copyright Tony Burns -->
<!-- cspell:ignore jetstream dynamicconfig signalwithstartworkflow signalworkflow workflowservice -->

# Appendix A. Evidence

Back to [the design](README.md).

Each claim here was read from `main` of a public repository on 2026-11-21, except the NATS and Temporal sections, read on 2026-11-22. Two items under [awrel today](#awrel-today) have other sources: the webhook payload line cites a live awrel delivery and octokit's published examples, both read on 2026-11-22, and one observation, marked Unverified, comes from a coordinator note outside public repositories. Paths are relative to that repository.

## awrel today

The read was taken at tbhb-dev/awrel commit `2c2ff52`, which is the merge of the Grafana alerts route (awrel#121). The latest tag is `v0.7.1`. Sessions run the client at `v0.6.0` or later. **Unverified:** session state files exist on the coordinator host. That comes from a coordinator note, not in a public repository. The state file format arrived in `v0.6.0` (awrel#62).

### Where it runs

- awrel is a Cloudflare Worker with one SQLite-backed Durable Object named `Relay`, written in TypeScript: `src/index.ts`, `src/relay.ts`, `src/connect.ts`, `src/github.ts`, `src/grafana.ts`, `src/env.ts`.
- It runs on Cloudflare's hosted platform, not on an owned host. It has no container, no systemd unit, and no tunnel. `wrangler.jsonc` defines a staging and a production environment, and custom domains carry ingress.
- `.github/workflows/deploy.yml` deploys staging and then production on each push to `main` from GitHub-hosted runners. It tests the new version at 0% traffic, cuts over, and rolls back on failure.
- `src/index.ts` routes `POST /webhooks/github`, `POST /webhooks/grafana`, and `GET /connect`, and answers 404 for anything else.
- The Go client lives in the same repository under module `go.tbhb.dev/awrel` (`go.mod`), with `cmd/awrel` and packages under `internal/`: `protocol`, `relayclient`, `mcpserver`, `delivery`, `compact`, `persist`, `logfile`, `transport`, `buildmeta`.

### Protocol

From `docs/reference/relay-protocol.md`:

- The handshake is `GET /connect` with a bearer token and the subprotocol `awrel.v1`.
- Each message is one JSON text frame. Client messages are capped at 4096 bytes.
- Client messages are `subscribe`, `unsubscribe`, `list`, `echo`, and `ping`. Server messages are `subscriptions`, `error`, `echoed`, `pong`, and `event`.
- The event envelope is `{type, provider, event, delivery, repository, payload}`.
- A subscription names `provider`, and for GitHub both `event` and `repository`. Optional filters are `action`, `conclusion`, `name`, and `workflow_path`. All filters must match, a list matches any member, and strings compare exactly.
- **Proven** by a read of `src/protocol.ts` at awrel commit `6793e94`, a descendant of the commit above: each filter has a table of the events it supports, and a filter on another event gets `bad_subscription`. List filters take 1 to 8 strings, deduplicated, and `name` takes 1 to 128 printable ASCII characters. `issues`, `issue_comment`, `pull_request`, and `pull_request_review` have no required filter.
- **Proven** by awrel's delivery for wiki#73 on 2026-11-22 and by octokit's published examples (`payload-examples/api.github.com/` in `octokit/webhooks`, read 2026-11-22): `pull_request.number` and `pull_request.labels[].name` on `pull_request` and `pull_request_review/submitted`, `issue.number` and `issue.labels[].name` on `issues/labeled`, `issues/unlabeled`, and `issue_comment/created`, and `label.name` on `labeled` and `unlabeled`. **Unverified:** whether a real `unlabeled` delivery still lists the removed label in `issue.labels`, since the example does. Settling it needs a live delivery, which means removing a label from an item.
- `check_run`, `workflow_run`, and `workflow_job` need `name` or `conclusion`, and `check_suite` needs `conclusion`. `action` works only for events in an explicit table.
- Subscriptions live in each socket's attachment, with a 12,288-byte budget. `Relay.publish` in `src/relay.ts` fans out to every matching socket from one Durable Object instance.
- The client pings every 30 seconds with a 10-second timeout, and redials with jittered backoff from 1 to 60 seconds (`internal/relayclient/relayclient.go`). On reconnect it replays its subscriptions.

### MCP client

- Tools are `subscribe`, `unsubscribe`, `list_subscriptions`, and `check_delivery` (`internal/mcpserver/mcpserver.go`). It exposes no resources.
- Events reach the session as channel notifications through the experimental `claude/channel` capability. Claude Code needs its development channels flag.
- Payloads are compacted by default (`internal/compact`), and payloads over 64 KiB become a short note.
- Configuration comes from `AWREL_ENDPOINT`, `AWREL_CONNECT_TOKEN` or `AWREL_CONNECT_TOKEN_FILE`, `AWREL_SESSION_ID`, and `AWREL_PERSIST`.
- Subscriptions persist per session key under the XDG state directory, mode 0600, pruned after 14 days.

### Auth and secrets

- GitHub webhooks are checked with HMAC-SHA256 over the raw body in `X-Hub-Signature-256` (`src/github.ts`). A `ping` event is answered and not relayed.
- Grafana alerts are checked with a hex HMAC-SHA256 in `X-Grafana-Alerting-Signature`, in constant time (`src/grafana.ts`). Bodies over 1 MiB get 413, and `alerts` must hold 1 to 100 entries.
- Sessions share one static bearer token, compared in constant time (`src/connect.ts`). There's no per-client identity.
- Worker secret names are `GITHUB_WEBHOOK_SECRET`, `GRAFANA_WEBHOOK_SECRET`, and `CONNECT_TOKEN`. A missing secret makes the Worker answer 503.

### Grafana route

`src/grafana.ts` publishes one event per firing or resolved alert that carries a scope label. The scope label maps to `kind/id`, which fills `repository`. The payload is an allowlist of alert name, status, scope, kind, value, start and end times, fingerprint, and link. The delivery ID is `gf-` plus a hash of the body and the alert's index, so a retried body keeps its IDs.

### Storage and telemetry

- awrel stores no events. The Durable Object's SQLite migration exists, but subscriptions live only in socket attachments.
- awrel has no OpenTelemetry. The Worker logs to the platform's log service, and the client writes structured logs without payloads.

### Known debt

- Events sent while a session is disconnected are lost, with no replay (`README.md`). The client emits a notice with the outage window.
- One shared token for every session.
- One Durable Object instance.
- A Claude Code `/mcp` reconnect leaves orphan client processes and double connections (`README.md`).
- Compacted payloads can still carry third-party text.
- Each new event needs a table update before `action` filtering works on it.

The open issue list wasn't read, because the read had no GitHub credentials.

## github-metrics receiver

tbhb-dev/github-metrics already receives GitHub webhooks through a tunnel on an owned host. The gateway copies its shape.

- Go with `net/http`: `internal/receiver/server.go`, `cmd/github-metrics/main.go`.
- `internal/receiver/receiver.go` checks `X-Hub-Signature-256` by streaming the body through HMAC-SHA256 under `http.MaxBytesReader` and comparing with `hmac.Equal`. It validates the event and delivery headers, answers 401, 400, or 413 on failure, and answers 202 only after the store call succeeds.
- `internal/store/store.go` keeps a queue and a dedupe table in one bbolt file. A reused delivery ID with a different body is an error.
- `deploy/compose.yaml` and `deploy/up.sh` run it under Docker Compose with an image pinned by digest, as user 65532, read-only root, all capabilities dropped, and no published port.
- A digest-pinned `cloudflared` sidecar on the same network carries the tunnel. It runs under `runc` because it can't resolve names under a sandboxed runtime, which the compose file records as debt. The tunnel routes only `POST /webhook` to the receiver, per `docs/architecture/decisions/0002-may-the-webhook-path-face-the-public-internet-th.md`.
- `internal/telemetry/telemetry.go` exports OTLP metrics when `OTEL_EXPORTER_OTLP_ENDPOINT` is set. Logs go to stdout as JSON. The container joins the collector's external network.
- Secrets are files read through `*_FILE` variables.

## Launcher

From tbhb-dev/tools, `src/tbhbdev/alpha/agent/` and `docs/reference/alpha/agent-configuration.md`:

- `tbhbdev alpha agent exec` takes a host, repository, model, effort, profile, run name, and prompt file. It starts a supervised headless run on the host. `wait`, `status`, `log`, and `diff` follow it.
- A launch records the calling session's terminal pane title. A reviewer launch is refused when that name is missing, so a non-interactive caller such as a Temporal worker needs a change in tools first.
- tools also holds `loop`, a shadow runner with a SQLite store of receipts and action keys. It's a likely consumer of gateway events.

## Telemetry library

No tbhb-dev/telemetry repository exists yet. The gateway starts with its own small telemetry package, as github-metrics does.

## NATS server and JetStream

Read on 2026-11-22 from shallow clones of nats-io/nats-server at `a0385c9` on `main`, whose `server/const.go` says `2.16.0-dev`, and nats-io/nats.go at `792e440` on `main`. The latest nats-server release tag is `v2.15.3`. Nothing here was run, so no claim is proven. Each line carries its label: existing means present in that source today, new means this design's choice, and unverified means not checked.

- **Existing.** `server.NewServer(*server.Options)` builds a server, and `Start`, `ReadyForConnections(time.Duration)`, `Shutdown`, and `WaitForShutdown` run and stop it (`server/server.go`).
- **Existing.** `server.Options` has `JetStream`, `StoreDir`, `DontListen`, `SyncAlways`, `SyncInterval`, `MaxPayload`, `JetStreamMaxMemory`, and `JetStreamMaxStore` (`server/opts.go`).
- **Existing.** With `DontListen` set, `Start` skips the client accept loop (`server/server.go`). `(*Server).InProcessConn` returns one end of a `net.Pipe`, and nats.go's `nats.InProcessServer` option connects through it instead of dialing TCP (`nats.go`).
- **Existing.** The file store's default sync interval is 2 minutes (`defaultSyncInterval` in `server/filestore.go`). The config value `sync_interval: always` sets `SyncAlways`. With `SyncAlways`, the file store calls `Sync` on the message block file after each write (`server/filestore.go`). A stream with one replica keeps `SyncAlways`, and a stream's async `PersistMode` turns it off (`server/stream.go`).
- **Unverified.** That the publish acknowledgement goes out only after that `Sync` returns, and the latency it adds per publish. Step 1 measures both.
- **Existing.** The stream config has `Name`, `Subjects`, `Storage`, `Replicas`, `MaxAge`, and `Duplicates`, in both the server and `jetstream.StreamConfig`. The default duplicate window is 2 minutes, cut to `MaxAge` when that is shorter, and a window longer than `MaxAge` is rejected (`server/stream.go`).
- **Existing.** The message ID header is `Nats-Msg-Id`. A repeated ID inside the window isn't stored, and the acknowledgement carries the original sequence and `"duplicate": true`. The server keeps the window's IDs in an in-memory map (`server/stream.go`).
- **Unverified.** The memory that map takes with a 7-day window at the gateway's event volume.
- **Existing.** With `JetStreamMaxMemory` unset, the server sizes JetStream memory at 75% of system memory, or of a lower Go memory limit, and with `JetStreamMaxStore` unset, at 75% of the free disk of the store directory (`dynJetStreamConfig` and `diskAvailable` in `server/jetstream.go`, read at `v2.15.3`).
- **Existing.** The default maximum payload is 1 MiB, and the server warns above 8 MiB (`MAX_PAYLOAD_SIZE` and `MAX_PAYLOAD_MAX_SIZE` in `server/const.go`).
- **Existing.** nats.go's `jetstream` package has `New`, `CreateStream`, `CreateOrUpdateConsumer`, `OrderedConsumer`, the publish option `WithMsgID`, and `Stream.GetMsg(ctx, seq)`. Consumer configs have `Durable`, `AckPolicy`, `MaxAckPending`, `FilterSubjects`, `DeliverPolicy`, and `OptStartSeq`, and the ordered consumer config has the last three. A message has `NakWithDelay`. The stream state has `FirstSeq` and `LastSeq` (`jetstream/`).
- **Unverified.** What an ordered consumer does when `OptStartSeq` is below the stream's first sequence. The design doesn't rely on it, since the server compares `resume_after + 1` with `FirstSeq` itself.
- **Unverified.** How a durable pull consumer behind the window moves when `MaxAge` deletes its next messages. The runner detects the jump in sequence either way.
- **Existing.** The `server.Options` fields and `Server` methods named here exist at the `v2.15.3` tag, and `defaultSyncInterval` there is 2 minutes (`server/opts.go`, `server/server.go`, `server/filestore.go`). The nats.go names weren't checked at a tag.
- **Existing.** A consumer's `DeliverPolicy` takes `DeliverAllPolicy`, `DeliverNewPolicy`, and `DeliverByStartTimePolicy` with `OptStartTime`, a consumer has `MaxDeliver` and `BackOff`, and a message has `Term` and `TermWithReason` (nats.go `jetstream/consumer_config.go` and `jetstream/message.go`).
- **Existing.** The server refuses an update to an existing consumer that changes its deliver policy, start sequence, start time, or ack policy, and doesn't list `FilterSubjects` among the fields it refuses (`checkNewConsumerConfig` in `server/consumer.go`). **Unverified:** where a durable consumer's position lands after its subjects widen. The routing design doesn't rely on it, since a new rule that needs a backfill gets a new consumer.
- **New.** The `EVENTS` stream, its subjects, the 7-day `MaxAge` and duplicate window, `SyncAlways`, the 6 MiB `MaxPayload`, and the body-hash header ([appendix C](C-protocol.md#stream-configuration)).

## Temporal

Read on 2026-11-22 from shallow clones of temporalio/sdk-go at `2650b5c` on `main`, temporalio/api at `f46ce49` on `main`, and two files of temporalio/temporal at `32ef011` on `main`: `common/dynamicconfig/constants.go` and `service/history/api/signalwithstartworkflow/signal_with_start_workflow.go`, with `service/history/api/signalworkflow/api.go`. The latest releases are sdk-go `v1.51.0` and temporal `v1.32.2`, and only the dynamic config file was read at `v1.32.2`. Temporal's documentation site wasn't read. Nothing was run, so no claim is proven. Labels follow the NATS section.

- **Existing.** `client.Dial(client.Options)` makes a client, and `Options` has `HostPort`, `Namespace`, `Credentials`, `Identity`, `DataConverter`, and `ConnectionOptions` with a `TLS` field. `client.NewAPIKeyStaticCredentials` and `client.NewMTLSCredentials` build credentials (sdk-go `client/client.go`, `internal/client.go`). The namespace defaults to `default`.
- **Existing.** The client has `ExecuteWorkflow(ctx, StartWorkflowOptions, workflow, args...)`, `SignalWorkflow(ctx, workflowID, runID, signalName, arg)`, and `SignalWithStartWorkflow(ctx, workflowID, signalName, signalArg, StartWorkflowOptions, workflow, workflowArgs...)`. Its comment says it signals a running workflow, or starts it and sends the signal in one transaction when it isn't running (sdk-go `client/client.go`).
- **Existing.** `StartWorkflowOptions` has `ID`, `TaskQueue` (required), `WorkflowIDReusePolicy`, `WorkflowIDConflictPolicy`, `WorkflowExecutionErrorWhenAlreadyStarted`, `RetryPolicy`, and the three timeouts. The reuse policy defaults to allow duplicate, including in `SignalWithStartWorkflow` (sdk-go `internal/client.go`).
- **Existing.** The reuse policies are `ALLOW_DUPLICATE`, `ALLOW_DUPLICATE_FAILED_ONLY`, `REJECT_DUPLICATE`, and the deprecated `TERMINATE_IF_RUNNING`. The conflict policies are `FAIL`, `USE_EXISTING`, and `TERMINATE_EXISTING` (api `temporal/api/enums/v1/workflow.proto`). The signal-with-start request carries both, with conflict policy `USE_EXISTING` by default, and has a `request_id` field (api `temporal/api/workflowservice/v1/request_response.proto`).
- **Existing.** The Go SDK's `SignalWithStartWorkflow` fills `request_id` with a fresh UUID, unless a Nexus operation's request ID is in the context under a key of the SDK's `internal` package, which other modules can't import. `SignalWorkflow` does the same. A caller can't set either request ID through the public client, short of calling the raw `WorkflowService()` client (sdk-go `internal/internal_workflow_client.go`).
- **Existing in `v1.32.2` and on `main`.** The server drops a signal-with-start whose request ID the running workflow already recorded (`IsSignalRequested` in `signal_with_start_workflow.go`, read at `v1.32.2`). On `main` the same drop sits under the dynamic config `history.enableSignalWithStartRequestIdDeduplication`, on by default, and only within the current run, since signal request IDs don't survive continue-as-new. The setting is absent from `v1.32.2`'s `constants.go`. A plain signal with a request ID the run has seen is dropped (`signalworkflow/api.go` on `main`).
- **Existing.** `UpdateWorkflowOptions` has `UpdateID`, unique per namespace, workflow ID, and run ID, and `WaitForStage`, which must be set (sdk-go `internal/internal_workflow_client.go`).
- **Existing.** The server's default per-event blob limit is 2 MiB (`limit.blobSize.error`), with a warning at 512 KiB (`limit.blobSize.warn`), both per namespace, in `main` and `v1.32.2`. On `main`, the history limits are 50 MiB and 51,200 events, and a workflow may have 2,000 pending outgoing signals to other workflows, a limit on its `SignalExternalWorkflowExecution` commands that doesn't bound incoming signals (`limit.numPendingSignals.error` in `constants.go`). The Go SDK's gRPC client allows messages up to 128 MiB by default (sdk-go `internal/grpc_dialer.go`), so the server's blob limit is the one that bites.
- **Unverified.** Which error type the server returns when a start or a signal-with-start meets a closed workflow under `REJECT_DUPLICATE`. Step 11b tests it against a dev server.
- **Existing.** `WorkflowIDConflictPolicy`, `WorkflowIDReusePolicy`, `WorkflowExecutionErrorWhenAlreadyStarted`, `NewAPIKeyStaticCredentials`, `NewMTLSCredentials`, `SignalWithStartWorkflow`, and `UpdateWorkflowOptions` all appear in `internal/client.go` at the sdk-go `v1.51.0` tag, read 2026-11-22 through the GitHub contents API. The other fields named in this section weren't checked at that tag, so they stay as read on `main`. In temporal `v1.32.2`, `common/dynamicconfig/constants.go` has no `EnableSignalWithStartRequestIdDeduplication`, which matches the claim that the server-side deduplication exists only on `main`.
- **New.** The `temporal` target, its rule actions, the reference argument, the error classes, and the `PARKED` stream ([appendix C](C-protocol.md#temporal-target)).
