# Availability SLOs for Streaming Responses

## Why this chapter exists

Classical availability is binary: the server returned `2xx` and the response was consumed, or it did not. On a streaming LLM endpoint, the binary breaks down. The user may have seen 200 tokens of a 2000-token response before the stream hung — is that request available? Available-and-degraded? Unavailable? The answer the SLO document gives *is a choice*, and making the choice explicit is the whole point of this chapter.

This chapter authors three distinct availability-adjacent SLIs — partial-success, graceful-degradation, and streaming-stall — any one of which by itself is weaker than the three together. The protocol-level `2xx-or-3xx` SLI from `mod-102` chapter 01 remains the floor, but on a streaming surface it is no longer the ceiling.

The reference for the classical framing is [SRE Workbook, Chapter 2 — Implementing SLOs](https://sre.google/workbook/implementing-slos/) and the right-tail-under-degradation framing is again Dean & Barroso's [*The Tail at Scale*](https://research.google/pubs/the-tail-at-scale/).

## 1. Why "the stream opened" and "the response completed" are not the same event

A streaming response passes through several states the SLI surface has to name explicitly:

1. **Request accepted.** The gateway returned 2xx and the server began generating.
2. **First token emitted.** The client's reader received bytes. From here the user believes the service is working.
3. **Steady-state generation.** Tokens are emitted at a reasonable rate. (TPOT and ITL from chapter 01 cover this.)
4. **Generation completed.** The server emitted the final token and closed the stream with a well-formed terminator.
5. **Client consumed the response.** The application-layer client read the stream, assembled the response, and the handling logic fired its downstream call / render.

A classical availability SLI only checks steps 1 and (sometimes) 5. On a streaming surface, a failure at step 2 (TTFT was so long the client timed out), step 3 (the stream stalled mid-flight), or step 4 (the stream was cut off without a proper terminator, truncating the response) are all *visible user failures* that look like success to a classical availability measurement.

The three SLIs below address those states.

## 2. SLI #1 — Partial-success rate

The fraction of streams that *completed cleanly* — reached step 4 above and emitted a well-formed terminator with the generation complete.

### Definition

- **Valid event.** Any streaming request that reached step 1 (accepted, 2xx returned, generation began). Excludes requests rejected at admission (those count against capacity SLIs, not availability).
- **Good event.** Valid event where (a) the stream was closed by the server (not the client), (b) the final chunk indicated completion (`stop` reason emitted, no abrupt termination), and (c) the server did not emit an error event mid-stream.
- **Bad event — the taxonomy.**
  - **Mid-stream error.** Server emitted an error event after one or more tokens. Classical availability sees 2xx; the user sees a broken response.
  - **Mid-stream truncation.** Stream closed without emitting the completion signal. Caused by scheduler preemption without resumption, model-runner crash on the specific sequence, or an upstream (provider) partial outage.
  - **Server-side timeout cut.** The response exceeded the server-side wall-clock budget and was forcibly terminated.

### Target shape

Partial-success is a near-availability SLI. Targets are high — 99.9%+ for a steady-state fleet. The failure modes above are not acceptable at any reasonable rate.

### Watch for

- **Client cancellation vs. server truncation.** These must be distinguished. A client that closes the connection (user navigated away, client-side timeout) is not a bad event for *this* SLI. The server-side signal distinguishes them; the SLI's valid-event filter encodes the distinction.
- **The `stop` reason.** Models can terminate for several reasons — reached end-of-sequence, hit max tokens, hit stop sequence, policy refusal. All of these are *good events* for partial-success (the stream completed as expected). Only an *unexpected* termination is a bad event.

## 3. SLI #2 — Graceful-degradation rate

The fraction of requests served from a *known-degraded path* when the primary path was unavailable, where "known-degraded" is explicitly defined in the policy document.

### Why this SLI exists

Model-backed services commonly have fallback paths: a second model provider when the primary returns 5xx, a smaller model when the primary is capacity-constrained, a cached response when the retrieval index is unavailable, a canned response when all above fail. From the user's perspective, a response from any of those paths is *not a classical failure* — they got an answer — but it is also not the response the primary path would have delivered. The SLI makes the fact that fallback fired *visible* to the error-budget stack.

### Definition

- **Valid event.** Any request that reached the service and was served (any path).
- **Good event.** Valid event served by the primary path at full quality.
- **Degradation events (the taxonomy).**
  - **Model fallback.** Primary model unavailable; secondary served the request. Quality may differ from primary.
  - **Provider fallback.** Primary provider unavailable; secondary provider served the request. (See `mod-107`.)
  - **Context truncation.** Original prompt too long for current capacity; request served with a truncated context.
  - **Retrieval fallback.** Retrieval index unavailable; request served without retrieval augmentation.
  - **Guardrail bypass.** A guardrail-sidecar failed open (safety / toxicity / PII filter unavailable); request served without that gate. (Fail-closed is also a choice; see `mod-110`.)
  - **Cached-response service.** A cached prior response returned instead of invoking the model.

### Target shape

Graceful-degradation is *not* binary good/bad. It is a *rate tracked for visibility and defensibility*. Typical SLO structure:

- **"Degraded serving ≤ X% of eligible requests over 28 days."** Target set by what the business accepts as "operating normally."
- **Individual degradation-type sub-SLIs.** Each category gets its own counter in the SLO document, so when the aggregate rate rises you can resolve which path is doing the degrading.

### Watch for

- **Tag every response with the path that served it.** Without the tag in the telemetry, you cannot compute this SLI. The gateway logs or the response headers are the right place; the OpenTelemetry [Generative AI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) include provider and model attributes you can extend with path-of-service.
- **Do not wrap this into partial-success.** A gracefully-degraded response completed cleanly and so it is a *good event* for partial-success — but it is *not* a good event for the fully-primary-path SLO. Keep the two counters separate.
- **Define fail-open vs. fail-closed per fallback.** A fail-open fallback counts as graceful degradation; a fail-closed fallback counts as partial-success failure. Decide per fallback and codify in the SLO document.

## 4. SLI #3 — Streaming-stall rate

The fraction of streams that *paused* mid-generation for longer than a threshold.

### Definition

- **Valid event.** Any streaming request that emitted at least one token (i.e., reached step 2 above).
- **Good event.** Valid event where no inter-token gap exceeded the stall threshold — typical starting threshold is 1 second, calibrated against the UI's visible stall tolerance.
- **Bad event.** Valid event containing at least one inter-token gap exceeding the threshold.

### Distinction from ITL

Chapter 01's ITL SLI measures the *distribution* of inter-token gaps as a latency property. The streaming-stall SLI treats "any gap exceeding threshold" as a *binary user-visible failure*. The relationship:

- ITL p99 may be 150 ms (great on average), but if 0.5% of streams contain a single 5-second gap, those users experience a visible stall.
- Streaming-stall rate puts that 0.5% directly into the error-budget stack rather than hiding it inside a latency distribution.

Both are useful; prefer publishing both. The stall rate is often the *first signal* on scheduler bugs (`mod-102` chapter 04 covers engine-side causes), provider partial outages, and KV-cache eviction cascades.

### Target shape

Streaming-stall is an availability-adjacent SLI with a high target — 99.9%+ of streams gap-free. It is the *right-tail-of-ITL* promoted to its own line in the stack. Fast-burn alerts on stall-rate detect production incidents earlier than ITL-distribution burn.

### Watch for

- **Threshold calibrated to UI.** 1 second is a sensible default for a chat UI; a voice UI may need 200 ms; a batch-processing pipeline may tolerate 10 s. Calibrate per workload-class and codify.
- **Network vs. server stall.** A stall at the client can be a network pause, not a server pause. Server-side telemetry is the authoritative source; the client-side measurement is for *reconciliation* and detection of gateway / egress issues.
- **Cumulative vs. maximum gap.** The SLI uses the *maximum* per-stream gap. A stream with ten 500 ms pauses is annoying but not a stall; a stream with one 5 s pause is a stall. Report both statistics but target the max.

## 5. The streaming availability triad in the error-budget stack

Together, the three SLIs above form the streaming-availability block of the multi-SLO stack from chapter 02. Added to the protocol-level `2xx-or-3xx` SLI, the availability surface for a streaming endpoint is:

| SLI | What it measures | Typical target |
|---|---|---|
| Protocol-level availability | 2xx-or-3xx on `/v1/chat/completions` | 99.9% |
| Partial-success | Streams reaching clean terminator | 99.9%+ |
| Graceful-degradation | Served by primary path at full quality | aggregate rate (e.g., ≥ 99%), with per-path sub-SLIs |
| Streaming-stall | No inter-token gap > 1 s | 99.9% |

Four SLIs. The surface is wider than classical availability because the failure surface is wider; the composition back into a single budget follows the sum-of-shortfalls arithmetic from chapter 02.

## 6. Walking a failure through the four SLIs

Four concrete failure cases make the SLI surface tangible.

### Case A — classical 5xx

Gateway returned 503 on a request. Protocol-level availability is bad. Partial-success is unaffected (the request never reached step 2). Graceful-degradation is unaffected unless this triggered a fallback for *subsequent* requests. Streaming-stall is unaffected.

**SLI diagnosis.** One SLI burns, three are clean. The on-call response is classical (infra incident).

### Case B — mid-stream server crash

Scheduler preempted the sequence and never resumed it; stream closed without the completion signal after 400 of 1000 tokens. Protocol-level availability is good (the request returned 2xx and streamed data). Partial-success is bad (truncation). Graceful-degradation is unaffected. Streaming-stall *may* be bad if there was a gap before the crash.

**SLI diagnosis.** Partial-success SLI isolates this. The on-call primary knows to look at scheduler state and crash logs on the engine, not at gateway-side 5xx counters.

### Case C — provider fallback

Primary provider started returning 5xx; LiteLLM / Portkey routed to the secondary provider. Request completed cleanly on secondary; user got a response. Protocol-level availability is good (2xx). Partial-success is good (clean completion). Graceful-degradation *is bad* (served by secondary). Streaming-stall is good.

**SLI diagnosis.** Only graceful-degradation SLI burns. The on-call primary is routed to the provider-fallback runbook (`mod-107`) rather than to a false infra-incident path.

### Case D — scheduler-induced stall

A noisy co-tenant arrived and caused a 4-second ITL gap mid-stream; the stream recovered and completed. Protocol-level availability is good. Partial-success is good. Graceful-degradation is unaffected. Streaming-stall *is bad*.

**SLI diagnosis.** Only streaming-stall SLI burns. The on-call primary knows to look at scheduler interference and admission policy (`mod-102` chapter 04 territory) rather than at completion logs.

The point of the four cases: the richer SLI surface *diagnoses by burn pattern*. A classical single-SLI surface would show 2xx-rate good in three of the four cases and the on-call would be blind to the user-visible failure.

## 7. Measurement where streaming meets the rest of the stack

Three measurement decisions to codify in the SLO document.

### Server side vs. client side

As with the latency SLIs from chapter 01, the streaming availability SLIs can be measured server-side or client-side. Pick the server side for the operator-facing number (lower latency to signal, directly controllable); supplement with client-side where you have the telemetry (user-perceived truth).

If server-side and client-side diverge substantially, that divergence is itself a signal — often a gateway / egress / client-SDK issue. Track the delta as its own low-priority SLI.

### Sampling vs. 100% measurement

Partial-success and streaming-stall should be measured on 100% of traffic; they are inexpensive counters derived from existing stream telemetry. Graceful-degradation is also 100% (tag every response with the path). None of these three are sampled; quality SLIs from chapter 02 are the sampled ones.

### Window alignment with the quality SLIs

All four availability-block SLIs use the 28-day rolling window to align with the quality SLIs and simplify the error-budget-stack view. A shorter window (7d) amplifies noise on the sparser signals (graceful-degradation is often rare); 28d is the right default.

## Summary

- **"Available" is not binary on a streaming surface.** The user experiences the stream in states; the SLI surface names them. Protocol-level 2xx is the floor, not the ceiling.
- **Three streaming-specific availability SLIs** — partial-success (did the stream complete cleanly?), graceful-degradation (was it served by the primary path?), streaming-stall (did it pause longer than threshold?).
- **Partial-success** tracks stream completion and distinguishes server-side truncation from client cancellation.
- **Graceful-degradation** promotes fallback-serving into the error-budget stack with per-path sub-SLIs. The graceful part is that the user still got an answer; the degradation part is that the SRE knows which fallback path served it.
- **Streaming-stall** promotes the right-tail of ITL into its own user-visible availability SLI; it is where scheduler-interference and provider-degradation incidents surface earliest.
- **Walking failures through the four SLIs** diagnoses by burn pattern — a 5xx burns one SLI, a provider fallback burns a different SLI, a scheduler stall burns yet another. The SLI surface is the first-pass triage ladder.

The next chapter authors the fourth SLI dimension — cost — and ties it to business unit economics rather than treating it as a free dashboard number.
