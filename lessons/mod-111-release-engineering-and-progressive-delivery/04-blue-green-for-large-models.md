# Blue-Green for Large Models

## Why this chapter exists

Canary (chapter 01) and shadow (chapter 03) presume the candidate deploy can be run at a fraction of the primary's capacity without an operational problem. For most model-backed services, that holds. For a slice of them — the ones where the model does not fit on a single node, where loading the weights takes tens of minutes, where KV-cache paging crosses NVLink, where the serving layer runs a tensor-parallel group of six to eight GPUs per replica — it does not.

On those fleets, the question "can we run the new model alongside the old one?" becomes a resource question with a hard constraint. The candidate's weights need N GPUs; the primary's weights need N GPUs; the cluster has 2N (or less) because the capacity plan from `mod-106` chapter 05 was sized to run one of them. Shipping a model update then becomes a **cutover**, not a ramp: at the moment the new model is routed to, the old model has to free its capacity so the new model can occupy it, and vice versa on rollback.

Blue-green is the pattern that handles this. It is older than canary — the term predates progressive delivery — and its discipline (dual deploy, atomic routing switch, old deploy retained for rollback) is well-settled in classical SRE. The AI SRE's extensions are three:

- **Dual-load is often not possible.** The classical blue-green assumption that both environments can run full production traffic at the same time breaks on large models. The cutover must be done without ever having both deploys serving full load.
- **The cutover is not atomic in the way the classical pattern assumes.** A model can be serving in-flight streaming requests that take 30–120 seconds; the switch to the new model has to decide what to do with them.
- **Client observability of model version matters.** On classical services, the client does not care which replica served the request. On an AI service, the client may want to know which model served it — for logging, for cost attribution, for debugging divergent behaviour, for compliance.

This chapter writes the AI-adapted blue-green, with the three extensions explicit. The chapter's boundary: the engine-side drain and in-flight swap primitives (how a vLLM / TensorRT-LLM / TGI worker actually finishes its in-flight requests before releasing its KV cache) are at the `ai-infra-inference-engineer` peer. This module owns the **orchestration contract** — what the traffic layer and the control plane do to coordinate the drain and the swap. Chapter 08 §1 draws the boundary.

## 1. The classical blue-green, in one page

Two production-sized environments run in parallel:

- **Blue.** The currently serving environment. All production traffic.
- **Green.** The candidate environment, running the new version. No production traffic (or synthetic / smoke-test traffic only).

The release is a two-step operation:

1. **Deploy green.** Build the new image / weights / config, provision the green environment, run smoke tests, warm caches.
2. **Switch.** Flip the ingress / load balancer / mesh to route all production traffic to green. Blue remains running for a rollback window.

The virtues are atomicity and reversibility: at any moment, exactly one environment serves production, and the switch back to the previous environment is a single config change.

For a stateless web service the pattern is cheap. Two environments at half the per-request cost add up to the same cost; dual-load is a few hours; the switch is a mesh config.

## 2. Where blue-green breaks on large models

Three forces push against the classical pattern on large-model fleets.

### 2.1 Dual-load capacity

For a model whose serving replica takes 4–8 GPUs and whose production footprint is tens of replicas (hundreds of GPUs), running blue and green at full production capacity simultaneously doubles the GPU footprint. On a provider-rate-limited or budget-rate-limited fleet, that doubling is not available; on a hosted-GPU cluster, procuring 2× for a release window is operationally expensive. The practical constraint is often **1.1×–1.3× available for the release window**, not 2×.

The AI-adapted shape is:

- **Green is provisioned at the minimum capacity to serve the cutover** (e.g., warm-start with a smaller replica count sized to initial load after cutover, autoscale up as traffic rises).
- **Blue is drained before green is at full capacity** so the sum never exceeds the available GPU budget.
- **The autoscaler is set for an unusually fast scale-up on green** at the moment of cutover, because the warm-start capacity is not enough for peak.

### 2.2 Model-load time

A model whose weights are 70–400 GB takes minutes to load from object storage into GPU memory. Even with weight-sharded parallel loading (tensor parallel across the TP group), a cold replica is not ready to serve for 2–10 minutes. Shadow traffic to a cold green replica measures the cold-start path, not the steady-state path.

The discipline:

- **Pre-warm green.** The green environment is deployed, loaded, and warmed (synthetic prompts, cache priming) *before* the cutover window opens. The cutover reads green as a hot, pre-warmed environment.
- **Smoke-test the warmth.** A pre-cutover health check includes a handful of real-shape smoke prompts to verify TTFT and TPOT match the SLO, not just liveness.
- **Pre-warmed KV cache or prompt cache are explicit deliverables.** Many provider and self-hosted stacks implement prompt caches; the cache is cold at green startup and warm only after traffic flows. A green deploy whose first hour is uncached degrades the TTFT contract for the first hour; §5 writes the drain-and-swap discipline that accounts for this.

### 2.3 In-flight streaming request longevity

A streaming request may remain open for 30–120 seconds (or longer on long-context agent routes). A cutover that routes new requests to green at `T=0` must decide what to do with blue's requests still open at `T=0+N`.

Three options:

- **Hold at current — drain blue.** Blue continues to serve its in-flight streams; new requests go to green; blue decommissions when its stream count hits zero. The drain window is bounded by the longest streaming request's duration.
- **Cut immediately — blue aborts in-flight.** Clients see mid-stream failures. Not acceptable for user-visible routes; sometimes acceptable for internal routes if the failure mode is documented.
- **Dual-send — cutover but keep blue as fallback for a window.** Each new request's primary is green, with an automatic fallback to blue on green error. This holds dual-load longer than §2.1's budget allows and is used only when neither drain nor abort is acceptable.

The default for a user-visible route is drain: §5 writes it.

## 3. Pre-cutover work — the hot green

Before the cutover switch is flipped, the green environment must be:

- **Deployed and healthy.** Readiness probes green.
- **Pre-warmed.** Weights loaded, first-token latency measured against a smoke-test set, within the TTFT SLO.
- **Pre-populated for caches** where possible. The prompt cache may need a corpus of recent prompts replayed; the retrieval index may need a warmup query stream; the KV-cache behaviour under the fleet's long-context pattern may need a scripted warmup.
- **Covered by its own SLO dashboard.** The green deploy emits SLIs with `version=green`; the SRE watches green's dashboard as independently as blue's.
- **Signed off by the two-predicate release gate** (chapter 05). Both eval-gate and SLO-gate are green; promotion is approved.

The pre-cutover work is often more time than the cutover itself. A 70B-model replica with 4 GPUs and weight-sharded loading may take 6–10 minutes per replica to warm; a fleet of 20 replicas is 20 × 6 = 120 replica-minutes total, which parallelises depending on the pull-bandwidth from object storage to each node. This is mod-106 territory and the chapter 06 of mod-106 capacity plan includes a "release warmup window" allocation.

## 4. The atomic routing cutover

The cutover is a routing-layer operation. Three mechanisms, in order of preference:

### 4.1 Mesh-level `VirtualService` edit

For an Istio-backed fleet, the cutover is a single `VirtualService` edit:

```yaml
# before
http:
  - route:
      - destination: { host: chat-serving-blue }
        weight: 100

# after
http:
  - route:
      - destination: { host: chat-serving-green }
        weight: 100
```

The mesh propagates the edit to all gateways / sidecars within seconds. The cutover is atomic per-request: any request that arrived before the propagation hits blue; any after hits green; no half-state exists at the request granularity.

This is the preferred mechanism because the cutover is a declarative artifact in Git, reviewed and audited.

### 4.2 Load-balancer target group swap

For cloud-native load balancers (AWS ALB target groups, GCP backend services, Azure Load Balancer backend pools), the cutover is a target-group switch. Slightly slower propagation than mesh, often a single control-plane call. The audit shape is less clean than mesh (the LB console shows the current state, not the history).

### 4.3 DNS flip

A DNS record change from blue to green. Bluntest and slowest (DNS TTL propagation measured in seconds to minutes). Appropriate only when the mesh / LB options are unavailable and the fleet can tolerate a DNS-TTL-bounded window of mixed routing. A fleet with a 5-second DNS TTL and in-flight streaming requests of 60 seconds should not use DNS flip.

### 4.4 The cutover log

Regardless of mechanism, the cutover emits a structured event:

```yaml
cutover:
  started_at: 2026-02-15T14:00:00Z
  completed_at: 2026-02-15T14:00:03Z
  route: chat
  from: chat-serving-blue  (manifest sha: ab12cd34)
  to:   chat-serving-green (manifest sha: ef56ab78)
  mechanism: istio-virtualservice-edit
  operator: j.ferguson
  approvals:
    - release-gate-contract-v1: pass
    - eval-gate-verdict: pass (chapter 05)
    - slo-gate-verdict: pass (chapter 05)
    - canary-pre-stage: pass (if applicable; chapter 05 §6)
```

The event is written to the audit log and the fixture ledger (`mod-108` chapter 05). Chapter 06 reads it to compute deployment frequency.

## 5. Drain-and-swap discipline

The cutover routes **new** requests to green at `T=0`. The question is what to do with blue's in-flight streams.

### 5.1 The drain contract

The drain contract specifies, per fleet, the behaviour of blue after the cutover. The standard shape:

- **Drain period.** The maximum wall-clock time blue holds in-flight streams after the cutover. For most consumer-facing routes, 5 minutes is long enough to clear ≥99.9% of streams; for agent / long-context routes, 15–30 minutes may be needed.
- **In-flight request handling.** Blue continues to serve its open streams to completion; blue refuses all new connections (achieved by the mesh cutover); blue emits a `draining=true` flag in its health check.
- **Hard-abort threshold.** If an in-flight request has not completed by the end of the drain period, it is terminated (connection reset or SSE error event). The user sees a mid-stream error; the fleet's `rb-streaming-stall` runbook (`mod-108` chapter 01) describes the failure shape.
- **Decommission.** After the drain period, blue's replicas scale to zero (or the deployment is left at `replicas=0` for the rollback window; see §6).

### 5.2 Engine-side drain primitives

The drain is enforced at the inference engine. Engines vary:

- **vLLM.** The `/v1/health` endpoint can be used to signal drain; `--request-cancellation-aware` request handling is a configuration knob. The engine supports cancelling in-flight requests on signal.
- **TGI / TensorRT-LLM / NVIDIA Triton.** Each has its own drain path; the relevant documentation is at the engine vendor.

The specifics are the `ai-infra-inference-engineer` peer's domain (chapter 08 §1). This module's contract is: **the inference-engine peer exposes a drain primitive the orchestration contract can invoke**. The AI SRE does not write the engine's drain code; it specifies the behaviour the orchestration depends on.

<!-- needs-research: confirm the exact drain-signal flag / endpoint names for vLLM, TGI, TensorRT-LLM, and Triton in the publication window; specifics rotate across releases -->

### 5.3 The stuck-stream fallback

A stream that cannot drain (a tool call blocked on an external slow dependency, a long-context agent that is still running at the hard-abort threshold) is a reliability event. The runsheet:

1. **Blue drops the stream** with a structured error event the client's SSE handler can recognise.
2. **The client retries** (per the fleet's retry policy) and the retry hits green.
3. **The incident ledger records** the stuck-stream rate per cutover.

A cutover that routinely produces stuck-stream incidents is a cutover whose drain period is too short, or whose route has a longer-tailed streaming distribution than the drain contract allows for.

## 6. Rollback — the symmetric cutover

Blue is kept at `replicas=0` (or minimum-ready-state) for a **rollback window** after the cutover. The rollback is a symmetric cutover: route traffic back to blue, drain green, decommission green.

The rollback window is a policy decision:

- **Short rollback (24 h).** Enough to catch catastrophic regressions; cheap to maintain blue at `replicas=0`.
- **Long rollback (7 days).** Enough to catch slow-burn regressions on business metrics (chapter 01 §3.4); more expensive if blue's weights still occupy object storage and the retention window extends.
- **Indefinite rollback.** The previous model is permanent; green becomes blue's successor only when the next release's green is ready. Appropriate for the most risk-averse fleets.

The rollback cutover follows the same mesh-level operation as the forward cutover. The key discipline:

- **Rollback is tested.** A dry-run rollback (re-point traffic to blue briefly, confirm the mesh edit works, point back to green) is performed within the rollback window to confirm the operation functions.
- **Rollback has its own runsheet.** `rb-blue-green-rollback` as a runbook-ID in the `mod-108` library. The runbook includes the mesh edit command, the drain-of-green behaviour, and the safety-gate check (chapter 05 writes the rollback condition).

## 7. The model-version header — client observability

A blue-green release that is invisible to the client is a release whose debug trail runs through the server-side log alone. For many AI fleets, client-side observability matters enough to be a contract:

- **Response header `X-Model-Version: <manifest-sha>`.** Every response from the serving layer includes a header identifying the model + config version that produced it.
- **Response header `X-Deploy-Environment: <blue|green>`.** During a cutover window, some clients (internal, admin, on-call) want to know which environment is serving them.
- **Response envelope field `model_version`** inside structured responses (JSON tool-call results) for durable logging.

Why the header matters:

- **Cost attribution.** The client-side logging infrastructure aggregates per-version cost without server-side joins.
- **Divergent-behaviour debugging.** A user's complaint about a changed response is debuggable immediately by the version tag.
- **Regulatory / compliance discipline.** Some jurisdictions require per-response model-identity disclosure (EU AI Act transparency obligations are the driving example — see `mod-110` chapter 03 for the compliance-telemetry contract; the specifics of per-response version disclosure depend on the jurisdiction's current regulations). <!-- needs-research: confirm which jurisdictions and risk-tiers require per-response model-version disclosure under the EU AI Act in its current enforcement timeline -->

The header is set by the serving layer, not by the client. The mesh may rewrite it but should not drop it.

## 8. The AI-adapted blue-green decision tree

Blue-green is not the default for all AI releases. The pattern is appropriate when:

- **Canary is not possible** because dual-load at the canary fraction + stable cannot be budgeted. This is rare on provider-backed routes (the provider absorbs it) and common on self-hosted large-model routes.
- **Shadow is not sufficient** because the pre-promotion signal set the fleet needs includes client-side latency, downstream behaviour, or user feedback — none of which shadow provides.
- **The model is large enough that in-place swap is not an option.** Updating a running replica's weights in place is at the inference-engine peer (chapter 08 §1); on engines that do not support it, the serving process must be restarted, and blue-green is the discipline that manages the restart cluster-wide.

The decision tree per release:

1. **Can the fleet run canary at 1% + stable in parallel?** If yes, canary first (chapter 01).
2. **Can the fleet afford shadow at any fraction?** If yes, shadow as a pre-canary stage (chapter 03).
3. **Can the engine do in-place weight swap?** If yes, use the engine's primitive and bypass blue-green.
4. **Otherwise, blue-green with the drain-and-swap discipline** of this chapter.

Many fleets use **both** blue-green and canary: blue-green between stable deployments (the engine-restart artifact-level step) and canary on top (the weight ramp as a routing-layer step within green's warm environment). The two compose without conflict as long as the capacity math holds.

## 9. The AI SRE's blue-green checklist

Before executing a blue-green cutover for the first time:

1. **Capacity is modelled.** `mod-106`'s calculator shows the GPU-minutes required for the cutover window, including pre-warm, drain, and rollback allowance.
2. **Green is hot before cutover.** Weights loaded, caches warmed, smoke tests passed, SLIs within SLO for ≥N minutes.
3. **The cutover mechanism is mesh-level** where possible, declarative, in Git.
4. **The drain contract is written** with a specific drain period, in-flight handling, and hard-abort threshold per route.
5. **The engine's drain primitive is verified** (not assumed) by the inference-engine peer.
6. **The `X-Model-Version` header is emitted** on every response.
7. **The rollback is tested** within the rollback window, not only in an abstract sense but by actually performing one in a staging-cluster dry-run.
8. **The `rb-blue-green-rollback` runbook is in the library.**
9. **The audit event** (§4.4) is written to the fixture ledger on every cutover and every rollback.
10. **The release-gate verdict** (chapter 05) is attached to the cutover record, both eval-gate and SLO-gate.

A cutover that happens without the ten is a cutover that produces a public post-mortem.

## Summary

- Blue-green on large models is the release pattern when canary and shadow cannot both run because the artifact is too large to dual-load at full capacity.
- The AI-adapted shape has three extensions over classical blue-green: dual-load is often not possible, the cutover is not fully atomic because streams are long-lived, and clients observe model version.
- Green is pre-warmed (weights loaded, caches primed, smoke tests passed) before the cutover; the pre-warm is often longer than the cutover itself.
- The cutover is a mesh-level `VirtualService` edit where possible, declarative and audited.
- Drain-and-swap discipline: blue continues to serve in-flight streams to completion for a bounded drain period, hard-aborts anything past the threshold, then decommissions.
- The engine's drain primitives are at the inference-engineer peer; this chapter specifies the orchestration contract that depends on them.
- The `X-Model-Version` header on every response is the client-side observability contract.
- Rollback is a symmetric cutover; blue is kept at `replicas=0` for a rollback window; the rollback is tested before being needed.
- Blue-green is selected per release by a decision tree; many fleets compose blue-green and canary.
- A ten-point checklist gates the first blue-green cutover per fleet.

Chapter 05 writes the release-gate contract that stitches canary, shadow, and blue-green together into a single promotion decision.
