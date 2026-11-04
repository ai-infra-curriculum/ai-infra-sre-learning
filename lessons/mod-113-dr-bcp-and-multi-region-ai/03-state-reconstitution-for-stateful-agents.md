# State Reconstitution for Stateful Agent Systems

## Why this chapter exists

The regional-outage playbook in chapter 02 moves traffic. It does not move state. On a classical REST service the distinction is thin — the state is in a database that is already cross-region replicated by the database. On an AI agent system the state surface is wider, the replication semantics are not uniform across its parts, and the "state" the user cares about includes things the transactional-database vocabulary does not name: the conversation history, the retrieved context, the in-flight tool-call stack, the trace record an upstream request will try to read, and (sometimes) the KV cache that an in-flight stream was mid-generation against. A failover that moves traffic without moving these produces an agent that, from the user's perspective, "lost its memory in the middle of the conversation."

This chapter writes the state taxonomy, the per-class replication strategy, and the RPO / RTO targets that turn the taxonomy into a design. It stays substrate-agnostic where it can — the discipline is the same whether the conversation store is Redis, DynamoDB, Cloud Spanner, or a Postgres row — and names the vendor primitives only where the primitive materially changes the design.

Primary references: [Amazon DynamoDB global tables](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GlobalTables.html), [Amazon Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html), [Cloud Spanner multi-region configurations](https://cloud.google.com/spanner/docs/instance-configurations), [Google Cloud SQL high availability](https://cloud.google.com/sql/docs/postgres/high-availability), [Azure Cosmos DB global distribution](https://learn.microsoft.com/en-us/azure/cosmos-db/distribute-data-globally), [Azure SQL active geo-replication](https://learn.microsoft.com/en-us/azure/azure-sql/database/active-geo-replication-overview), [Redis Enterprise Active-Active Geo-Distribution](https://docs.redis.com/latest/rs/databases/active-active/), the [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem) and the [PACELC theorem](https://en.wikipedia.org/wiki/PACELC_theorem) as framing, and `mod-104` chapter 03 for the trace-store discipline this chapter extends to cross-region.

## 1. The state taxonomy

The function names six distinct state classes on an AI agent system. Each has a distinct ownership story, a distinct replication story, and a distinct failover story. A plan that treats them as one thing will under-replicate the high-sensitivity classes and over-replicate the low-sensitivity ones.

### 1.1 Conversation / memory state

The ordered sequence of user and assistant turns that constitutes the "conversation." Also called "memory" when the system accumulates it across sessions.

- **What depends on it.** Every next turn's context assembly. The user's expectation that "the agent remembers what we just said."
- **Volume.** Per-conversation, tens of KB to low MBs on long conversations; per-tenant, aggregated across conversations, up to TBs.
- **Access shape.** Read-dominant on the hot path; append-only writes on each turn.
- **RPO sensitivity.** High. A failover that loses the last conversation turn is user-visible as a dropped turn. Target: seconds to a single turn.
- **RTO sensitivity.** Moderate. If the store is unreadable for 30 s during failover, the session stalls; the user retries and expects it to work.

### 1.2 KV cache state

The attention key-value cache sitting in GPU memory for in-flight generations. Sometimes persisted to a host-side or distributed cache for prefix reuse (`mod-104` chapter 06).

- **What depends on it.** The in-flight stream. A warm cache accelerates subsequent requests whose prefix matches.
- **Volume.** Per-request, GBs in GPU memory; per-fleet, the entire HBM footprint of the serving pods.
- **Access shape.** Ephemeral. GPU-local writes, GPU-local reads.
- **RPO sensitivity.** Low to zero. A KV cache that is lost on failover is replayed from scratch; the cost is latency and recomputation, not correctness.
- **RTO sensitivity.** Low. The warm cache's benefit is latency optimisation, not functional correctness.

The function does *not* replicate the GPU-resident KV cache. The distributed prefix cache is replicated only if the latency budget demands it (§3.2 below).

### 1.3 Tool-call state

The in-flight tool invocation stack for an agent's turn — "the agent called `search_docs`, is awaiting the result, will call `summarise` next." Sometimes called the "scratchpad" or "working memory."

- **What depends on it.** Resuming or re-planning the agent after a failure that interrupted a tool call. The idempotency contract for external side effects (`mod-105` chapter 05).
- **Volume.** Per-request, KBs to low MBs; per-fleet, modest.
- **Access shape.** Written on each tool-call transition; read on resume.
- **RPO sensitivity.** Mixed. A tool call that was issued externally (a payment, a database write) must be reconciled on recovery, not re-issued blindly. A pure-compute tool call can be re-issued.
- **RTO sensitivity.** Moderate to high. A failover that drops tool-call state turns every in-flight agent into a decision: "replay, resume, or abort."

### 1.4 Retrieval context / RAG snapshots

The retrieved documents that were in the context window at generation time. `mod-112` chapter 02 owns the retrieval-index reliability; this class is the *per-request snapshot* of what was retrieved.

- **What depends on it.** Reproducibility of the response. Audit — "what documents did the model see?" (`mod-110` chapter 02). Judge-score investigation on a regression.
- **Volume.** Per-request, KBs to tens of KBs; retained with the trace per `mod-110` chapter 05.
- **Access shape.** Written once per request; read only on investigation.
- **RPO sensitivity.** Low. The user does not notice loss. The audit record does; retention policy applies.
- **RTO sensitivity.** Low. Does not block serving.

### 1.5 Trace store

The OTel traces, Prometheus time series, and structured logs the serving tier emits (`mod-104` chapters 01–03).

- **What depends on it.** Incident investigation. Post-mortem reconstruction. Audit. Compliance retention.
- **Volume.** Large. Weeks-to-months of retention in a hot tier, longer in cold storage.
- **Access shape.** Write-dominant from serving; read on investigation, dashboard, and audit.
- **RPO sensitivity.** Moderate. A trace gap during a DR event is itself a compliance concern; the audit retention target is defined in `mod-110` chapter 05.
- **RTO sensitivity.** Low. The trace store can be unreadable for minutes during a failover and the serving tier keeps running; the function pays attention to the buffering story that keeps emission alive during the outage.

### 1.6 Configuration / registry state

The prompt registry (`mod-112` chapter 01 §3), the runtime-config CRDs (`mod-112` chapter 01 §4), the feature-flag platform (`mod-112` chapter 01 §4.3), the model-registry pointers (`mod-111` chapter 01). The "what to serve" state, distinct from the "what was served" state.

- **What depends on it.** Every request. If the secondary region cannot resolve the current prompt or config, it cannot serve.
- **Volume.** Small (KBs to MBs).
- **Access shape.** Read-dominant on the hot path; writes are promotion events.
- **RPO sensitivity.** High on writes — a promotion that reaches primary and not secondary is split-brain.
- **RTO sensitivity.** High. Secondary cannot serve without the current version.

### 1.7 The per-class summary

| Class | RPO target | RTO target | Default replication | Where it lives |
|-------|------------|------------|---------------------|----------------|
| Conversation / memory | seconds to one turn | tens of seconds | Multi-region database or managed cache with active geo-replication | §2 |
| KV cache (GPU-resident) | n/a | n/a | Not replicated | §3.1 |
| Distributed prefix cache | best-effort | best-effort | Not replicated by default; reconsidered only for latency reasons | §3.2 |
| Tool-call state | per-request, exactly-once on external effects | seconds | Multi-region KV / database + idempotency keys | §4 |
| Retrieval context snapshot | retention-bounded | best-effort | Replicated with the trace | §5 |
| Trace store | buffered; minutes | minutes | Cross-region write pipeline with a local buffer | §5 |
| Configuration / registry | zero on primary promotion; seconds on secondary | seconds | GitOps + registry service with multi-region deployment | §6 |

The targets are defaults; a service's ISCP (chapter 06 §3) sets the per-service numbers from the BIA.

## 2. Replicating the conversation / memory state

The conversation store is the state class that is both the most user-sensitive and the one with the most mature vendor primitives. The function picks a substrate per workload and inherits most of the replication mechanics from the vendor.

### 2.1 The substrate choice

The common choices and the shape each imposes:

- **Managed multi-region KV / document stores.** DynamoDB global tables (active-active, last-writer-wins), Cosmos DB (configurable consistency from strong to eventual, active-active in multi-region write mode), Spanner (external consistency across regions), MongoDB Atlas global clusters. The function's default on new builds; the replication is the vendor's problem, the failover is a connection-string change.
- **Managed SQL with geo-replication.** Aurora Global Database (primary + read-only secondary; ~1 s typical replication lag; failover in minutes), Azure SQL active geo-replication (same shape), Cloud SQL cross-region read replicas. Promotes a secondary to primary on failover; writes against the old primary during the gap are lost.
- **Self-managed Redis or Postgres with custom replication.** Redis Enterprise Active-Active ([CRDT-based](https://docs.redis.com/latest/rs/databases/active-active/)), Patroni-managed Postgres, self-managed Redis clusters with Sentinel. More operational surface; usually not the default unless the function's team already owns the operational discipline.

The function cites CAP / PACELC when choosing: a multi-region conversation store that chooses "consistency" during a regional partition is unavailable during the partition; a store that chooses "availability" is available but may produce split-brain writes that need reconciliation. For agent conversation state the function's default is **availability + last-writer-wins with session affinity** — the user is unlikely to be writing from two regions at once, so the LWW semantics are acceptable, and the availability gain on the single-session hot path is real.

### 2.2 Session affinity and sticky routing

The simplest way to avoid conversation-state write conflicts is to not generate them. The endpoint-failover substrate (chapter 02 §2) is configured so that a user's session sticks to one region as long as that region is healthy. On failover:

- The user's subsequent requests resolve to the healthy region.
- The healthy region reads the conversation store and finds the state (because the store is cross-region replicated).
- The user's experience is continuity; the conversation has "all the memory up to and including the last confirmed turn."

The failure mode the sticky-routing contract prevents: a user refreshes their browser, hits the secondary during a partial outage, their write lands on the secondary, then the primary recovers and their next request lands on the primary with stale state. Session affinity by user / session cookie (or by device identifier at the mobile edge) is the mitigation.

### 2.3 Read-after-write semantics across regions

A conversation turn written in region A and immediately read from region B must return the written value on a timescale the user expects (sub-second). The vendor's documented replication lag is the lower bound; the application's observed lag is what the function measures. The function emits:

- `conversation_store_replication_lag_seconds{src_region, dst_region}` — vendor-reported or synthetically measured.
- `conversation_store_read_after_write_p99_seconds{route}` — measured from the serving tier's own read-after-write probe.

The probe is a canary: write a well-known key in primary, read it from the secondary on a loop, record the time to resolution. Deviation from steady state is an early signal of a replication-path problem.

### 2.4 Reconciliation on split brain

When session affinity is broken by failover and the function's chosen consistency class allows writes on both sides of a partition, the application must reconcile. The reconciliation policy is written per service:

- **Append-only, union on recovery.** Conversation turns are append-only; divergent turn sets from the two sides are unioned on recovery, ordered by timestamp, deduplicated by turn hash. The user may see turns the agent "didn't say" from their perspective; the ISCP cites this as an acceptable degradation at the residency / reliability trade-off (chapter 04 §3).
- **Primary-wins, replay secondary.** The primary's conversation state is authoritative on recovery; writes that landed only on the secondary are replayed through the primary. Suitable when traffic was largely drained to the secondary and its writes are small.
- **Manual reconciliation for long-tail splits.** When the volume of divergent writes exceeds a threshold, an operator queues a reconciliation batch job. Rare; triggered by extended multi-hour DR events.

The reconciliation policy is cited in the ISCP; the DR drill (chapter 05 §4) exercises at least one reconciliation path annually.

## 3. KV and distributed cache state

### 3.1 The GPU-resident KV cache is not replicated

The decision is categorical. The attention KV cache for an in-flight generation is tied to the specific GPU on the specific pod on the specific node and is invalidated by any pod restart. Replicating it cross-region would require GPU-to-GPU replication at latency incompatible with any reasonable budget. The function does not do it.

The consequence is that **in-flight streams terminate on failover**. The client-side contract is: stream termination is a defined failure mode; the client retries from the beginning of the current turn; the server-side serves from the new region with no in-flight stream state. The agent's conversation-state replication (§2) is what makes the retry resumable; the KV cache is not.

### 3.2 The distributed prefix cache

Prefix caching across pods — the pattern `mod-104` chapter 06 describes, where a shared KV cache store amortises the prefix cost across requests — is operationally different. The cache is persisted in a host-side or external store (Redis, a bespoke KV service). Replicating it cross-region is possible; the function does not do it by default because:

- The cache is a latency optimisation; a cold cache in the failover region is slower but correct.
- Cross-region replication of cache contents consumes bandwidth that is better spent on weights.
- The cache's access pattern (every request writes) does not survive the write amplification of cross-region replication at tier-1 load.

The function warms the secondary's prefix cache with shadow traffic (chapter 02 §4.3), not with replication. If a workload's latency SLO cannot absorb the cold-cache cost on failover, the function reconsiders at the ISCP level; most do not need the replication.

## 4. Tool-call state and continuity

The tool-call state surface is the one where "DR for AI" diverges hardest from "DR for web services." The surface composes with the idempotency discipline from `mod-105` chapter 05 — the agent's external-effect integrity depends on both.

### 4.1 The state the function persists

Per in-flight agent turn:

- **The turn identifier.** The content hash of the user's input + the conversation prefix + the model version + the prompt hash + the config hash; the same key used for idempotency.
- **The tool-call plan.** The ordered list of tool invocations the agent has produced so far in this turn.
- **The tool-call status per step.** `pending`, `in-flight`, `completed-with-result`, `failed`.
- **The idempotency key per external effect.** Written before the tool call is issued, not after.
- **The step's observable result.** Written on completion.

This state is small and read-dominated. The function stores it in the same substrate as the conversation state or in a separate, lighter KV store with the same multi-region replication posture.

### 4.2 The three-way decision on recovery

When the secondary picks up a turn that was in-flight in the primary at failover, the agent's control plane reads the tool-call state and chooses:

- **Replay.** The tool call did not reach the external service (status `pending`, no idempotency record on the external side). Issue the call from the secondary. Safe for all tool classes.
- **Resume.** The tool call completed and returned a result (`completed-with-result`). The agent continues from the next planned step without re-issuing.
- **Reconcile and resume.** The tool call was issued and the external side has a record for the idempotency key (`in-flight` in local state, resolvable on the external side). Query the external side with the idempotency key, resolve to the known result, write it to local state, and continue.

The policy is per tool, cited in the tool's integration doc. A tool with no idempotency support (and no safe replay) is a tool the function refuses to invoke without an operator-in-the-loop on resume; this is a design constraint upstream of DR, surfaced in the DR plan.

### 4.3 The streaming-mid-tool edge case

A failover that hits an agent mid-generation on a tool-call-producing stream has a composite failure: the stream terminates (per §3.1), the in-flight tool-call state may or may not have been persisted (depending on when the agent control plane checkpoints). The function's convention: **checkpoint the tool-call state at the moment of tool-call emission, not at completion**. The state is written synchronously before the tool is invoked. The resulting worst case is a tool call the state reflects as `pending` that the external side never saw — safe, replayable.

### 4.4 Long-running agent runs

An agent run that spans minutes (deep research, multi-step planning) is particularly exposed to DR events. The function treats long-running runs as explicit stateful workflows:

- Named checkpoint points at each tool-call boundary.
- Persisted run identifier, consumable across regions.
- Operator-facing resume CLI that reads the run state and continues from the last checkpoint.
- A DR drill (chapter 05 §3.4) that mid-run triggers a failover and measures the resume outcome.

Durable-execution frameworks (Temporal, AWS Step Functions, Google Cloud Workflows) model this explicitly; the function selects one or writes the equivalent on top of its agent runtime. The choice is out of scope for this module; the DR contract is what this module writes.

## 5. Retrieval context and trace-store replication

### 5.1 Retrieval context snapshot

Per-request retrieval snapshots are emitted on the trace. They are retention-bound (per `mod-110` chapter 05), not sensitivity-bound. The function's default: snapshots are stored with the trace, cross-region replicated along with the trace, and are not separately replicated with lower latency. A gap in retrieval-snapshot availability during a DR event is accepted and documented.

### 5.2 Trace-store write pipeline

The trace-store write pipeline is the function's primary concern because its reliability determines whether a DR event is *investigatable* after the fact. The pattern:

- **Local buffering.** The serving tier emits spans to a local OTel collector (`mod-104` chapter 01). The collector has a persistent-queue buffer sized for a DR-event duration (hours, not minutes).
- **Regional ingest.** The collector forwards to a regional ingest service (vendor-specific; Tempo, Jaeger, a hosted vendor). The ingest service is within the primary region.
- **Cross-region export.** The ingest service forwards processed spans to the cross-region trace store. This is where the replication happens, asynchronously and bandwidth-governed.

On a regional DR event:

- The primary region's collector continues to buffer locally.
- The regional ingest may be down; the collector's buffer absorbs the gap.
- When the primary is recovered, the buffer drains to the restored ingest.
- The secondary region's trace emission continues normally.

The function monitors:

- `otel_collector_queue_depth{region, type}` — buffer utilisation.
- `trace_ingest_lag_seconds{region}` — time from span emission to ingest-visibility.
- `trace_cross_region_lag_seconds` — time from primary ingest to secondary resolvability.

A buffer that overflows during a DR event is a trace gap; the function sizes it against the ISCP's RTO target plus a margin.

### 5.3 The trace store as evidence

For regulated workloads, the trace store is the evidence pack. `mod-110` chapter 02 writes the audit discipline; chapter 06 §5 of this module writes the regulator-facing summary scaffold. The DR contract here is: **the trace emission must survive a DR event**. A failover that loses trace coverage of the DR event itself is a reporting problem on top of the operational one.

## 6. Configuration / registry state

The configuration surface is small but has the strictest RPO / RTO because every request reads it.

### 6.1 Replication primitives

The function's defaults:

- **GitOps for the config CRDs.** Both regions' clusters pull from the same Git repository via Argo CD or Flux. A config change is a commit; the controller applies it to all target clusters. The RPO is the Git-to-cluster sync interval (seconds to a minute); the RTO is the controller's convergence time.
- **Multi-region deployment of the prompt registry service.** The registry is a service; it has its own fleet in each region, backed by a shared storage layer (a cross-region-replicated object store or a multi-region database). A prompt promotion writes to the shared storage layer, which replicates to the secondary before the secondary's registry serves the new hash.
- **Flag-platform vendor's multi-region story.** If the function uses a hosted flag platform (LaunchDarkly, Unleash), the platform's own geo-replication applies. The function reads the vendor's documentation and includes it in the ISCP's vendor-dependency table.

### 6.2 Fail-safe serving on registry unreachability

A secondary region whose registry is unreachable during a DR event must not crash the serving tier. The function's convention: the serving tier caches the last-known-good resolution of each (route, channel) pair with a long TTL (hours) and refuses to invalidate the cache if the registry is unavailable. The behaviour is documented in the registry's own ISCP as the "static-mode" posture.

### 6.3 Promotion discipline during DR

During an active DR event, the function pauses non-urgent promotions. The rationale: a region with impaired consistency is not the region to ship a prompt / config / model change through. The DR plan explicitly cites this hold; the release engineering runbook (`mod-111` chapter 05 §6) is aware of the DR-event hold flag.

## 7. The RPO / RTO matrix

The function's deliverable at the end of this chapter is a per-service, per-state-class RPO / RTO matrix. A worked example for a single tier-1 chat service:

| State class | RPO target | RTO target | Substrate | Reconciliation policy |
|-------------|------------|------------|-----------|-----------------------|
| Conversation / memory | 1 turn | 30 s | DynamoDB global tables | Append-only, union on recovery |
| GPU KV cache | n/a | n/a | — | Stream terminates; client retries turn |
| Distributed prefix cache | best-effort | best-effort | — | Cold on failover; shadow-warm |
| Tool-call state | zero on external effects | 15 s | DynamoDB global tables (same as conv.) | Three-way (replay / resume / reconcile) |
| Retrieval context snapshot | 5 min | n/a | Trace store | None (write-once) |
| Trace store | 2 min | 10 min | OTel → regional ingest → cross-region sink | Replay from local buffer |
| Prompt registry | zero on primary; 30 s on secondary | 30 s | Multi-region registry + GitOps | Primary-wins; serving tier static-mode |
| Runtime-config CRDs | zero on primary; 60 s on secondary | 60 s | GitOps + Argo CD | Primary-wins |

The matrix is cited in the ISCP (chapter 06 §3) and reviewed annually per the BIA (chapter 06 §4).

## 8. The state-reconstitution checklist

Before enabling DR for a service, the function confirms per state class:

1. **The class is named** (§1.1–§1.6) and mapped to a specific store.
2. **The vendor / substrate's replication primitive** is cited.
3. **The RPO and RTO targets** are documented and read from the BIA.
4. **Session affinity / sticky routing** is wired where cross-region-write split-brain is possible.
5. **A read-after-write probe** emits a cross-region lag SLI (§2.3).
6. **A reconciliation policy** is cited for every class where a split brain is plausible.
7. **The three-way tool-call recovery decision** is implemented for every tool with external side effects.
8. **The trace-store buffer** is sized against the DR-event RTO target plus a margin.
9. **Registry static-mode behaviour** is implemented in the serving tier.
10. **The RPO / RTO matrix (§7)** is present in the ISCP and reviewed annually.

A service that does not pass the checklist does not have "state replication"; it has "a database that happens to be in two places."

## Summary

- Six state classes compose the DR surface of an AI agent system: conversation / memory, GPU KV cache, distributed prefix cache, tool-call state, retrieval context snapshot, trace store, and configuration / registry state. Each has its own RPO / RTO sensitivity and its own replication strategy.
- Conversation / memory state is the user-sensitive surface. The default posture is availability + last-writer-wins with session affinity and a documented reconciliation policy. Cross-region vendor primitives (DynamoDB global tables, Cosmos DB, Spanner, Aurora Global Database) carry most of the weight.
- The GPU KV cache is not replicated; in-flight streams terminate on failover and the client retries. The distributed prefix cache is not replicated by default; it is warmed in the secondary with shadow traffic.
- Tool-call state is persisted at the moment of tool-call emission and drives a three-way recovery decision (replay, resume, reconcile) backed by the `mod-105` chapter 05 idempotency discipline. Long-running agent runs run under a durable-execution framework.
- The trace store survives a DR event via local buffering in the collector, regional ingest, and cross-region export. The function sizes the buffer against the ISCP RTO plus margin and emits SLIs on buffer depth and cross-region lag.
- Configuration / registry state replicates via GitOps and multi-region service deployment. The serving tier runs a static-mode cache on registry unreachability. Non-urgent promotions are held during an active DR event.
- The per-service RPO / RTO matrix (§7) is the artifact the ISCP cites and the drill programme (chapter 05) validates.

Chapter 04 adds the constraint the replication choices above cannot ignore: data residency. Not every substrate is legally allowed to serve every user's state, and the failover direction may be constrained by regulation before it is constrained by availability.
