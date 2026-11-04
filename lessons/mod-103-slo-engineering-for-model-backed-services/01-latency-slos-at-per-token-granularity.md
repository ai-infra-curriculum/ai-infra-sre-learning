# Latency SLOs at Per-Token Granularity

## Why this chapter exists

The classical latency SLO is one number per request — "p99 ≤ 300 ms on `/v1/search`." For a streaming LLM endpoint that number is not meaningful. A user watching a chat response does not experience a single request-latency; they experience *when the first token appeared*, *how fast the words came out after that*, *whether the stream ever stalled*, and *when it finished*. Those are four different user experiences and each one of them is a different SLI.

This chapter authors the per-token latency SLI surface — TTFT, TPOT, ITL, E2E, and goodput — then shows how to pick the right percentile targets for each workload class (chat, batch, RAG, tool-calling, agentic, multimodal) and how to defend the resulting numbers to a non-engineering stakeholder. The downstream payoff is that once each of these SLIs is in writing, the on-call rotation from `mod-102` chapter 03 has specific things to burn against and the policy from chapter 05 of this module has specific numbers to halt releases on.

The classical reference for the SLI framing is [SRE Workbook, Chapter 2 — Implementing SLOs](https://sre.google/workbook/implementing-slos/). The right-tail-latency framing that per-token SLIs are built on comes from Dean & Barroso, *The Tail at Scale* ([CACM, 2013](https://research.google/pubs/the-tail-at-scale/)).

## 1. The five per-token latency SLIs

### TTFT — time to first token

The elapsed wall-clock time from when the client's request was accepted by the gateway to when the first output token is emitted on the response stream.

- **What it measures.** The *perceived start* of the response. For a chat UI, this is the delay before the user sees the cursor start typing. For a tool-calling agent, it is the delay before the orchestrator can start parsing the first JSON chunk and routing to a tool.
- **What it does not measure.** Anything after the first token. A TTFT-only SLO can be clean while the actual stream is slow or stalls — see TPOT and ITL below.
- **What drives it.** Prefill compute (which depends on prompt length), queueing at the scheduler, prefix-cache hit vs. miss, admission control, and the number of concurrent sequences in flight competing for prefill capacity. Covered engine-side in `mod-102` chapter 04 and capacity-side in `mod-106`.
- **Where to measure it.** Server-side at the engine for the operator-facing number (vLLM / SGLang / TRT-LLM / Triton all expose this via `/metrics`; the OpenTelemetry Semantic Conventions for Generative AI name the analogous attribute [`gen_ai.server.time_to_first_token`](https://opentelemetry.io/docs/specs/semconv/gen-ai/)). Also measure client-side at the gateway or the client SDK for the user-facing number — it includes network and gateway overhead the engine-side number does not.
- **How to report.** Percentile over a window, broken out by prompt-length bucket. A single aggregate TTFT percentile hides the fact that TTFT for a 500-token prompt and a 32k-token prompt are different animals.

### TPOT — time per output token

The steady-state generation rate: the mean inter-token wall-clock gap *after* the first token.

- **What it measures.** The *throughput of the stream* from the user's perspective, in tokens per second. It is the rate at which the UI fills in once it starts.
- **What drives it.** Decode-step latency, which depends on model size, quantization, batch size (larger batch increases throughput but typically slows per-stream TPOT via interference), and memory-bandwidth contention on the GPU. Decode is memory-bound; TPOT regresses when HBM-bandwidth headroom is lost (long-context attention, large batch).
- **Why "mean" and not percentile per token.** TPOT is itself an aggregate (tokens / elapsed for the generation window). Alongside it, report ITL below for the *stability* of the stream.
- **How to report.** The per-request TPOT is one number — mean inter-token gap over the generation window. The SLO is a percentile over requests: "p95 TPOT ≤ 60 ms/token." Also bucket by output-length distribution; TPOT on a 20-token completion and a 2000-token completion average different things.

### ITL — inter-token latency

The per-token gap, as a distribution, inside a single request.

- **What it measures.** The *jitter and stall-resistance* of the stream. TPOT can be clean (mean 50 ms/token) while ITL has a long right tail (one 2-second gap buried in a 400-token response). The user experiences the 2-second gap as "the model froze."
- **What drives it.** Scheduler interference (another request's prefill arriving mid-decode), preemption, KV-cache eviction, upstream network bursts, and GPU-side events (DCGM throttling, XID interrupts).
- **How to report.** Two statistics in the SLO: (a) the per-request p99 ITL — "p99 inter-token gap ≤ 150 ms for 99% of requests," and (b) the *stall rate* — "fewer than 0.1% of requests contain any inter-token gap over 1s." The second one is the stall SLI and it should be its own line, not buried inside the ITL percentile.

### E2E — end-to-end request latency

Total wall-clock time from request accepted to final token emitted (or structured-output envelope closed).

- **What it measures.** The *total time the user waits*. For non-streaming endpoints it is the only latency SLI. For streaming endpoints it is the bound on how long a long response can take overall, which is what batch pipelines and offline agents care about.
- **What drives it.** TTFT + output-length × TPOT, roughly, plus whatever tail-latency events occurred mid-stream. Long-tail E2E is almost always driven by ITL stalls or prefill queueing on a loaded fleet.
- **How to report.** Percentile over requests, bucketed by output-length distribution. An SLO of "p99 E2E ≤ 10s" on a corpus where 10% of requests ask for 4000-token responses is a different SLO than the same number on a 50-token-response corpus.

### Goodput — successful tokens per second per replica, under SLO

Goodput is throughput-constrained-by-SLO: the sustained rate of *useful* output tokens (not error tokens, not retries) that a single serving replica produces while *simultaneously* staying within the TTFT / TPOT / ITL SLOs above.

- **What it measures.** The *capacity planning number*. Raw tokens/sec on a replica is the engine's benchmark number; goodput is what the fleet-sizing math from `mod-106` will actually consume.
- **What drives it.** Everything above, plus the batching knobs and the admission-control policy. Goodput is lower than raw throughput by construction — any configuration change that trades tail-latency for throughput moves the two in opposite directions.
- **How to report.** As a scalar per model + replica-shape pair, derived from a load test or steady-state production measurement. It is less a user-facing SLO and more a *platform-internal contract* that the latency SLOs imply — "if the TTFT SLO is 500 ms, a replica of this shape delivers X goodput before the SLO starts breaking." The distinction between an SLI (user-facing, on the error-budget stack) and a derived contract (platform-internal, consumed by capacity planning) is useful to keep clean.

The DistServe paper (Zhong et al., 2024 — ["DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving"](https://arxiv.org/abs/2401.09670)) is the canonical treatment of goodput as an LLM-serving metric; read it alongside `mod-102` chapter 06 on NVIDIA Dynamo for the serving-system side of the picture.

## 2. Picking percentiles — the honest framing

The classical answer to "which percentile?" is p99 for user-facing APIs, p99.9 for critical-path infrastructure. That answer is not wrong for LLM serving — but it is incomplete, because the *cost* of defending a tighter percentile is much higher under stochastic generation than under a CRUD workload.

Three forces to balance:

1. **User experience.** A p50 SLO ignores the users having a bad day; a p99.99 SLO ignores *almost nothing* at the cost of a dramatically larger headroom budget.
2. **Statistical power of the measurement.** On a low-traffic surface, p99.9 over a 28-day window may only admit ~1000 "bad" events per the budget — small sample, noisy signal. The burn-rate alerting from [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) is the discipline here.
3. **Stochastic-output interaction with latency.** On an LLM, long completions and short completions have fundamentally different latency profiles; a global p99 is dominated by the long tail of output lengths. Prefer bucketed percentiles over aggregate.

A sensible default authoring pattern:

- **p95 as the primary user-facing percentile for streaming endpoints.** Catches real user pain without over-anchoring on small-sample tail events.
- **p99 as the paging threshold.** Fast-burn and medium-burn alerts fire against p99; this is how you learn about the tail without putting the tail in the SLO target.
- **p99.9 only for critical-path non-streaming endpoints** — structured-output / tool-calling where a stalled stream fails a downstream contract.

## 3. Latency targets by workload class

There is no single right number. There *is* a set of workload shapes that have been publicly discussed in the engineering literature, and each has a plausible target range. Treat the following as *starting points for the SLO review conversation with the business sponsor*, not as fixed values.

### Chat (interactive single-turn or short multi-turn)

- **User contract.** Perceived responsiveness; the user is watching the cursor.
- **Target shape.** TTFT is the dominant SLI — a long TTFT reads as "the service is slow." TPOT matters second — the stream needs to be at least human reading speed (around 150–200 words per minute, or roughly 3–5 tokens per second minimum).
- **Illustrative starting range.** TTFT p95 in the few-hundred-ms to low-second range; TPOT p95 at a few tens of ms/token. The specific numbers depend on model size, context length distribution, and hardware. Anchor them to what your current fleet actually delivers at launch (baselining in chapter 06) and *then* negotiate tightening.
- **Dominant failure modes.** Prefill queueing under load, prefix-cache miss on cold prompts, scheduler interference from a bursty tenant on the same pool.

### Batch (offline inference, async jobs)

- **User contract.** Throughput, not responsiveness. The user submits a job and reads the result later.
- **Target shape.** TTFT is largely irrelevant — a 10-second queueing delay is fine. TPOT and E2E dominate; in particular E2E against job-size SLO ("95% of 10k-token summarization jobs complete in ≤ 2 minutes"). Goodput is the real planning number.
- **Dominant failure modes.** Scheduler starvation when the batch pool is co-tenanted with an interactive pool; capacity shortfalls when the batch submits exceed daily throughput.

### RAG (retrieval-augmented generation)

- **User contract.** User sees a chat UI with citations; same perceived responsiveness bar as chat, but with a retrieval step added to TTFT.
- **Target shape.** TTFT must include the retrieval latency — publish TTFT as *end-to-end to first generated token* so the number represents what the user sees, not what the LLM sees. Bucket by retrieval-cache-hit vs. miss because the two distributions are very different.
- **Dominant failure modes.** Vector store latency spikes, retrieval-cache miss on a cold query set, re-ranker model latency.

### Tool-calling (structured output + external tool invocation)

- **User contract.** The user does not see a stream; they see the *tool's* side-effect. "Reliability" is whether the tool fired with correct arguments.
- **Target shape.** E2E to *valid tool invocation* is the SLI. TTFT and TPOT are less user-visible; their role is to budget against the scheduler. Structured-output conformance (chapter 02) is composed with latency into the final SLO.
- **Dominant failure modes.** Parser retries (structured output invalid on first attempt), guardrail reject-and-retry, tool-API fallback latency.

### Agentic (multi-step, planner-executor, chain-of-thought with external calls)

- **User contract.** Task completion within a time budget. Individual LLM call latencies are a means, not an end.
- **Target shape.** The top-level SLO is on the *task*, not the call — "95% of task-of-type-X complete within N seconds." Internal per-call latencies (TTFT, TPOT) become *platform-internal contracts* consumed by the agent runtime, not user-facing SLOs. This is a reversal of the chat case.
- **Dominant failure modes.** Loop expansion (agent runs more steps than budgeted), recursive-tool failures that compound latency, planner-step quality drift that causes retries.

### Multimodal (image / audio / video input or output)

- **User contract.** Depends on the modality — audio TTFT is the latency to first audible phoneme; image generation is dominated by end-to-end to first intermediate render.
- **Target shape.** Define per-modality SLIs; do not reuse the text-side TTFT definition. For streamed audio, the first-audio-frame-ready metric is a separate SLI from text TTFT. For diffusion-style image generation, "time to first preview" is the proxy for TTFT.
- **Dominant failure modes.** Highly input-shape-dependent — a 10-second audio clip and a 10-minute audio clip have fundamentally different latency profiles; bucket aggressively.

Multimodal is the class the engineering literature has standardized on *least*. Treat any targets you author for it as explicitly provisional for the first SLO review cycle, with an explicit note that the eval peer and the modality owner will tighten them once baseline distributions are known.

## 4. Defending each threshold to the business sponsor

Every number in your SLO document will be asked about. The stakeholder in the room is usually not an SRE; they are the product owner, the business sponsor, or the executive signing the capacity budget. The defence you give has to translate.

A reusable defence template per SLI:

1. **What the user experiences at this threshold.** "At TTFT p95 = 800 ms, users on the chat surface perceive the response as starting *within one second*, which is the industry convention for 'instant.' At 1500 ms, users perceive a noticeable delay." Anchor to user-perceivable thresholds, not engineering-internal thresholds.
2. **What the business loses when this threshold is breached.** "A regression from 800 ms to 2000 ms TTFT correlates, in our product analytics, with a measurable drop in session completion." Be honest: if you do not have the correlation, say so and schedule the measurement as the next quarter's deliverable — do not fabricate.
3. **What it costs to deliver the threshold.** "At p95 TTFT = 800 ms with current fleet, we hold the threshold at *N* concurrent sessions; to hold it at *M* concurrent sessions requires *X* additional GPUs. Dropping to 1500 ms unlocks *M'* at zero added capacity." Cost is always a currency the sponsor understands.
4. **What the fallback is if we miss the threshold.** "When TTFT burns budget, the policy from chapter 05 freezes new model rollouts and prioritizes the remediation work. We do not degrade the user experience further; we halt the risk of worsening it."

Avoid the two anti-patterns:

- **"Industry standard is p99 ≤ 500 ms, so that is our target."** The sponsor cannot defend "industry standard" against the finance team asking why your GPU budget is growing 40% quarter over quarter. You can defend a number you have modelled to your workload and your traffic.
- **"This is the tightest we can deliver."** Possibly true at a point in time; irrelevant to the sponsor. The sponsor is buying a user experience, not a maximum feasible performance. Frame in user terms and in cost terms, not in engineering-feasibility terms.

## 5. Measurement gotchas worth writing into the SLI spec

A few specific traps that live inside the latency SLI definitions and bite teams in production. Enumerate them in your SLO document's "valid-event filter" section for each SLI.

- **Prefix-cache hits distort TTFT.** A request whose entire prompt hits the prefix cache will have an artificially low TTFT; your TTFT SLI should either exclude cache-hit requests from the sampled measurement or report cache-hit and cache-miss TTFT as separate SLIs. vLLM's [automatic prefix caching documentation](https://docs.vllm.ai/en/latest/features/automatic_prefix_caching.html) explains the mechanism; the measurement gotcha is yours to codify.
- **Chunked prefill bends TTFT.** With chunked prefill ([vLLM docs](https://docs.vllm.ai/en/latest/features/chunked_prefill.html)), the first "output token" may be emitted before the full prompt has been consumed, which can look like a TTFT improvement but is actually a decoupling of the measurement from the user's experience in some UIs. Decide, with the engine peer, which of the two definitions your SLI uses and lock it.
- **Speculative decoding complicates TPOT.** Rejected speculative tokens never reach the client but *did* consume decode steps. TPOT measured server-side will differ from TPOT measured client-side when speculative decoding is active. Pick one anchor side and stay consistent.
- **Streaming abandoned by the client is not a bad event.** If the user closed the tab after 200 tokens of a 1000-token response, the request did not fail — but a naive E2E measurement will record its latency against the SLI. Add a valid-event filter: "requests where the client disconnected mid-stream are excluded from E2E; they contribute to a separate abandonment SLI."
- **Retries from the client amplify tail latency.** If the client retries on an admission reject, the retry's latency rolls into the user-perceived latency but may be measured as a fresh request server-side. Decide the reconciliation at the gateway and codify it in the SLI.

Each of these should land as a single sentence in the valid-event-filter paragraph of the SLI, not as a footnote in a runbook. If a measurement convention is not in writing, the SLO silently drifts.

## Summary

- **Five latency SLIs** — TTFT, TPOT, ITL, E2E, goodput. Author each as a separate SLI with explicit measurement side (server / client), valid-event filter, and bucketing (prompt length / output length).
- **Percentiles** — p95 primary, p99 paging, p99.9 only for critical-path non-streaming surfaces. Bucket aggressively; a global p99 on streaming is dominated by output-length tail.
- **Workload class decides which SLIs matter** — chat is TTFT-dominant, batch is E2E / goodput-dominant, RAG folds retrieval into TTFT, tool-calling is E2E-to-valid-invocation, agentic measures task completion not call latency, multimodal is modality-specific and provisional.
- **Defend thresholds in user terms and cost terms**, not engineering-feasibility terms. The sponsor is buying a user experience against a capacity budget.
- **Write measurement gotchas into the SLI spec** — prefix-cache, chunked prefill, speculative decoding, client abandonment, retries. Conventions not in writing drift silently.

The next chapter composes these latency SLIs with quality SLIs into a single multi-dimensional error-budget stack.
