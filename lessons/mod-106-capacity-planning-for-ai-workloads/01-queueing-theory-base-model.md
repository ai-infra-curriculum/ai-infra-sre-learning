# The Queueing-Theory Base Model for Prefill-Bound and Decode-Bound Workloads

## Why this chapter exists

Capacity planning for a model-backed service is not capacity planning for a classical web service. A classical service treats a request as a single unit of work with one service time; `λ` (arrival rate), `μ` (service rate), and the number of servers `c` are enough to produce a working M/M/c model, and the model is accurate to the point of being useful for p99 latency projections. A model-backed service violates almost every M/M/c assumption at the same time: a single request has *two distinct phases* with different arithmetic (prefill and decode); service time for one phase is roughly proportional to input length while the other is roughly proportional to output length; requests are not served one-at-a-time but are *continuously batched* with other requests at different phases; portions of the computation are *cached* across requests; and newer serving strategies — speculative decoding, prefix caching, chunked prefill, disaggregated prefill/decode — add further coupling that the independence assumption of queueing theory does not survive.

That does not make queueing theory useless. It makes queueing theory the *first-order model* — the vocabulary an AI SRE uses to reason about where the fleet will break before building the simulator that answers the same question with 5% error instead of 50%. The first-order model is not optional; without it, every capacity conversation becomes "let's run more load tests" and headroom becomes a vibe. This chapter builds the first-order model, the shortest set of knobs you can turn to improve it before upgrading to simulation, and the crisp list of serving-strategy features that force the upgrade.

## 1. The two-phase shape of an LLM request

Every autoregressive LLM request has two phases with asymmetric arithmetic. The AI SRE must internalise this in the first chapter because every metric on the dashboard — TTFT, TPOT, ITL, queue depth, batch size — is interpretable only through this split.

### 1.1 Prefill

The prefill phase processes the prompt. Given `P` input tokens, the model runs *one forward pass on all `P` tokens in parallel*. The compute cost is approximately `2 · P · N` FLOPs per transformer parameter under a decoder-only model, where `N` is the parameter count — i.e., prefill is **compute-bound**. On modern accelerators, prefill saturates tensor cores; its throughput is reported in TFLOPs/s or in tokens/s with implicit linearity in `P`.

The user-visible signal of prefill is **TTFT** (time-to-first-token). The first token is emitted after the prefill forward pass completes and the first decode step is done. TTFT is dominated by prefill for prompts above a few hundred tokens; for very short prompts, TTFT is dominated by the engine's scheduling overhead (queue wait + batch admission + a small constant).

Prefill is **batchable across requests**: two concurrent requests with prompts of length `P1` and `P2` can be prefilled together with cost roughly proportional to `P1 + P2`, up to the engine's token-budget limit. Modern engines (vLLM, SGLang, TensorRT-LLM) chunk long prefills to interleave with decode (*chunked prefill*) so a single long prompt does not block decode progress for other requests.

### 1.2 Decode

The decode phase generates output tokens one at a time. Given `O` output tokens, decode runs `O` sequential forward passes, each on a single new token with access to all prior KV cache. Each decode step reads the full KV cache from memory; the compute per step is small compared to the memory traffic. Decode is **memory-bandwidth-bound**: the dominant cost is `KV-cache-bytes-per-token × O` reads from HBM.

The user-visible signals of decode are **TPOT** (time-per-output-token, average across decode steps) and **ITL** (inter-token-latency, per-step). Both are reported in milliseconds per token. On an H100 serving a 70B-parameter model in BF16, decode step time is on the order of 10–30 ms depending on batch size and context length; on H200 the same step is 20–40% faster because HBM bandwidth is higher at the same compute (see chapter 03 for the cross-generation table). These numbers are illustrative orders-of-magnitude for reasoning about the first-order model; your production engine's numbers must be measured, not derived from this chapter.

Decode is **batchable** across requests with the engine's continuous-batching scheduler — new requests can be admitted into the running batch between decode steps, and finishing requests leave. Batch size for decode is bounded by the KV-cache budget (HBM capacity minus model weights minus activation workspace) and by the engine's `max_num_seqs` configuration.

### 1.3 The prefill/decode split as a capacity concept

The split is not cosmetic. A chat workload with 200-token prompts and 500-token completions is *decode-bound*: most GPU-seconds are spent in decode, HBM bandwidth is the bottleneck, and adding more compute (a faster GPU with similar bandwidth) buys you less than adding more memory bandwidth (H100 → H200 or FP8 weights). A RAG workload with 8K-token prompts and 100-token completions is *prefill-bound*: most GPU-seconds are spent in prefill, tensor-core TFLOPs are the bottleneck, and the picture reverses. A batch summarisation workload with 32K-token prompts and 200-token completions is prefill-bound to an extreme — prefill dominates by more than 10× for a typical model.

An AI SRE classifies every surface the fleet serves as prefill-bound, decode-bound, or mixed. The classification is the first input to the forecast in chapter 02.

## 2. Little's Law as the first-order budget

Little's Law states that in a stable queueing system, the average number of items in the system equals the average arrival rate times the average time in system: `L = λ · W`. It holds under extraordinarily weak assumptions (the system is in steady state, work is conserved) and is unaffected by distributional details, which is why it is the one piece of queueing theory an AI SRE should reach for first.

For an LLM serving system under a request-per-second load `λ` and an end-to-end service time `W` (TTFT + decode time for the entire response), the average number of requests in the system — queued or in-flight — is `L = λ · W`. For continuous-batching engines, "in the system" means "either waiting for admission or in the running batch." If the engine's steady-state running batch size is `B̄`, and the queue-ahead admission policy is working, `L ≈ B̄` (queue depth stays small when the fleet is correctly sized).

### 2.1 How to use it

**Convert an SLO to a batch-size budget.** If the TTFT SLO is 2 s at p95 and the TPOT SLO is 50 ms, and the average completion is 500 tokens, average `W ≈ 2 + 500 · 0.05 = 27` s. If the target `λ` is 10 req/s per replica, average requests in the system is `L = 10 · 27 = 270`. The engine must run with a decode-batch large enough to absorb 270 in-flight requests on average; if `max_num_seqs=256`, the fleet will queue. Adjust `max_num_seqs` or add replicas.

**Convert a target throughput to a replica count.** If a fleet must sustain 500 tokens/s of decode output per replica and the target aggregate output is 50,000 tokens/s, the replica count is `50000 / 500 = 100`. The number must be padded for headroom (chapter 02 section 4) and for high-percentile latency (section 3 below). Little's Law gives you the mean; the mean is the bottom of the budget, not the top.

**Convert a queue depth signal to a scaling decision.** If sustained queue depth is `Q` and target in-system count is `L*`, and `Q` is drifting upward, either `λ` is rising or `W` is rising. Chapter 05's autoscaler reads `Q` as the signal and acts before TTFT visibly regresses.

### 2.2 What Little's Law does not tell you

Little's Law gives you the mean. It does not give you the p95 or p99. For AI workloads where completion length is heavy-tailed (chat with occasional long explanations; code-gen with occasional 2000-token outputs), the p99 of `W` can be 5–10× the mean, which means the p99 in-system count can be 5–10× the mean in-system count, which means the p99 queue depth is where the engine spills to the queue even though the mean is fine. The AI SRE uses Little's Law for the first-order sizing and then explicitly models the tail, either with M/M/c for distributional insight or with simulation (section 5).

## 3. M/M/c as a first-order model

M/M/c is the queueing model with Poisson arrivals, exponential service times, and `c` parallel servers sharing a single FIFO queue. It is wrong for an LLM serving system in detail — arrivals are not Poisson across a business day, service times are not exponential (they are heavy-tailed because completion length is heavy-tailed), and servers are not independent because they share the continuous-batching pool. It is still the first approximation worth doing, because it gives closed-form answers that calibrate intuition.

### 3.1 The utilisation ρ and the waiting trap

Let `ρ = λ / (c · μ)` be the utilisation, where `c · μ` is the fleet's aggregate service rate (requests per second across all replicas / batch slots) and `λ` is the arrival rate. For `ρ ≥ 1` the system is unstable and the queue grows without bound. For `ρ < 1`, the expected queue length under M/M/c grows as roughly `ρ² / (1 - ρ)` for `c = 1` (the Pollaczek–Khinchine intuition) and more gently for large `c`. The qualitative behaviour is the same: **as `ρ → 1`, waiting time blows up superlinearly**.

An AI SRE who has not seen this chart before should redraw it from memory. At `ρ = 0.70`, waiting time per request is small. At `ρ = 0.85`, it is 3× larger. At `ρ = 0.95`, it is 10× larger. At `ρ = 0.99`, it is 100× larger. **A fleet run at 95% utilisation is not "efficient"; it is one request away from a TTFT incident.** The standard capacity-planning headroom target of 50–70% peak utilisation comes directly from this curve.

The AI-specific wrinkle is that for a model-backed service, "utilisation" is not GPU utilisation (`nvidia-smi` percent). It is **effective utilisation of the scheduling resource that is actually bottlenecked** — either decode-batch slots (if decode-bound) or prefill-token-budget per step (if prefill-bound). GPU % utilisation routinely reads 95%+ on an idle decode engine (the SMs are busy on a small batch) and routinely reads 60% on a saturated prefill engine (compute is interleaved with KV-cache traffic). Chapter 05 returns to this when choosing the autoscaler metric.

### 3.2 Head-of-line blocking

M/M/c assumes a single FIFO queue and `c` independent servers. The LLM serving case is messier. If the engine admits requests into a single running batch, a single long-prompt request can occupy prefill slots for seconds, delaying every short-prompt request that arrives behind it in TTFT even though total throughput is fine. This is **head-of-line (HOL) blocking** at the prefill layer.

Mitigations the engine provides — the AI SRE should know they exist even though the engine peer owns them:

- **Chunked prefill.** The engine chunks a long prefill into fixed-size token chunks and interleaves them with decode steps for in-flight requests. Long prompts no longer block the batch; they slow it slightly.
- **Prefill/decode disaggregation.** Separate replicas serve prefill and decode. A request hits a prefill replica, KV cache is shipped to a decode replica, decode completes there. Long prefills are now fully isolated from decode latency; the cost is a KV-cache transfer (often over RDMA) and more orchestration.
- **Priority queues and request-length bucketing.** Classify requests at admission (short prompt, long prompt, premium tier) and schedule across multiple logical queues. Classical SRE pattern; the engine must support it.

An AI SRE's job is to notice when HOL blocking is biting — TTFT p99 regresses while TTFT p50 is fine and throughput is below the mean capacity — and route the fix to the engine peer.

### 3.3 Why `c` is slippery

In M/M/c, `c` is the number of servers. In an LLM serving system, `c` is not "the number of GPUs." The useful `c` depends on what you are modelling:

- **Prefill-bound fleet.** `c` is the number of prefill-concurrent token slots across the fleet — roughly `(prefill-tokens-per-step × replicas × steps-per-second)`. The arithmetic is messy; measurements are safer.
- **Decode-bound fleet.** `c` is the number of decode-concurrent sequence slots across the fleet — roughly `(max_num_seqs × replicas)`, bounded by KV-cache budget. Easier to measure directly.
- **Mixed.** `c` is two numbers, and the fleet is bounded by whichever is tighter for the current traffic shape.

The AI SRE reads the engine's scheduler metrics (`vllm:num_requests_running`, `vllm:num_requests_waiting`, `vllm:kv_cache_usage_perc`, or the equivalent in SGLang / TRT-LLM / TGI) rather than deriving `c` from hardware. The scheduler metric is `c`'s operational definition for this fleet.

## 4. Where the simple model breaks

M/M/c and Little's Law are useful as the first-order model. They get you inside 2× for sizing and inside the right order of magnitude for tail latency. They stop being useful when the engine uses any of the following, because each one violates an independence assumption baked into the queueing model.

### 4.1 Continuous batching

Continuous batching (used by vLLM, TGI, TRT-LLM, SGLang) admits new requests into the running batch between decode steps rather than waiting for the batch to finish. This is the defining feature of modern LLM serving; it violates the M/M/c assumption that servers are independent, because every in-flight request shares a single batched forward pass. A new short-decode request can delay every in-flight long-decode request by a fraction of a step time; a long request leaving the batch opens a decode slot that is filled immediately.

The impact on the model: service time per request is not drawn from a stable distribution — it depends on the current batch's composition. Simulation (section 5) captures this; closed-form M/M/c does not.

### 4.2 Prefix caching

Prefix caching (vLLM's radix cache, SGLang's RadixAttention, provider-side prompt caching from OpenAI / Anthropic / Google) stores the KV cache for prefix tokens shared across requests. A request whose prompt starts with a cached prefix can skip prefill for the cached portion. The service time is now a function of *the state of the cache*, which is a function of recent traffic, which violates the independence of consecutive requests — a burst of requests sharing a prefix is served much faster than a uniform-over-prompts load with the same arrival rate.

The impact on capacity planning: nominal prefill throughput understates effective throughput by the prefix-cache hit rate. If your workload has a 70% prefix-cache hit rate and you planned capacity on 0% hit rate, you are over-provisioned by roughly `1 / (1 - 0.7) = 3.3×` on prefill compute; if you planned on 90% hit rate and your prompt template changed overnight (chapter 05 of `mod-105`), you are under-provisioned by the same factor. Chapter 02 forces the hit-rate forecast into the sizing exercise so this does not happen by accident.

### 4.3 Speculative decoding

Speculative decoding runs a smaller draft model to propose `k` tokens ahead, then verifies them against the main model in parallel in a single forward pass. If the acceptance rate is `α`, effective tokens per step is `1 + k · α`, and decode throughput per sequence rises by that factor. The engine peer tunes `k` and the draft model; the AI SRE must know that **a fleet running speculative decoding has a different effective decode throughput than one not running it**, and that the acceptance rate drifts with the input distribution (new topic area, new language, new prompt style → `α` drops). Capacity forecasts that assume a fixed `α` will miss on traffic-shape shifts.

### 4.4 Chunked prefill and disaggregated prefill/decode

As noted in section 3.2: these change the shape of the service-time distribution without changing its mean. Chunked prefill smooths out p99 TTFT at modest throughput cost. Disaggregated serving turns one fleet into two coupled fleets with a KV-cache transfer step between them; capacity planning now has two node pools, a KV-transfer bandwidth budget, and a non-trivial failure mode (a decode replica going down strands prefill work). Both need simulation to size well.

### 4.5 Multi-LoRA serving

A single base model serving multiple LoRA adapters (vLLM's MultiLoRA, S-LoRA) has per-adapter service-time overhead and shared KV-cache budget. From the queueing-theory lens, the fleet is one pool with request-dependent service time. Simulation again.

## 5. When to upgrade to simulation

The upgrade signal is operational: your capacity model's p95 projection is off from measured p95 by more than 30%, consistently, in the same direction, after calibrating `c` and `W` from production. The model has the wrong shape, not the wrong constants.

### 5.1 What a usable simulator looks like

For AI SRE purposes, a serving-system simulator is a discrete-event simulation that models:

- **Arrival process.** Request arrivals with a configurable distribution (Poisson, MMPP for bursty, time-of-day modulated).
- **Request shape.** Prompt length and completion length drawn from distributions that match your production histograms.
- **Prefix-cache hit distribution.** Per-request probability of hitting a prefix in cache, with a model of cache eviction.
- **Engine scheduler.** A simplified version of the continuous-batching scheduler with `max_num_seqs`, prefill-token budget per step, and KV-cache budget.
- **Hardware model.** Per-step time as a function of batch composition (prefill tokens this step, decode sequences this step, context length distribution across in-flight requests). Calibrated against engine benchmarks, not theory.

Public simulators: **LLMServingSim** (academic prototype from KAIST), **vidur** (research simulator from Microsoft Research, calibrated against vLLM), **AlpaServe** (older but still a useful template). For AI SRE use, the choice is: adopt one, calibrate against your fleet, and keep the calibration living.

### 5.2 What simulation is not

Simulation is not a replacement for load testing. The simulator estimates how the fleet will behave; load testing confirms that it does. Chapter 06 of this module closes the loop: run periodic load tests, update the simulator's calibration from the results, re-run the forecast.

Simulation is also not a replacement for observability. If your prefix-cache hit rate panel is broken, your simulator's hit-rate input is garbage. The observability stack from `mod-104` is the simulator's data feed.

## 6. The AI SRE's queueing-theory worksheet

The chapter's first-order model, condensed to a checklist the on-call can actually use:

1. **Name the workload's phase profile.** Prefill-bound, decode-bound, or mixed. If mixed, note the split (e.g., 60% decode-GPU-seconds, 40% prefill-GPU-seconds).
2. **Measure `W`.** End-to-end request service time: TTFT + decode time. Produce a distribution histogram, not a mean.
3. **Measure `λ`.** Request rate, stratified by workload surface (chat vs. RAG vs. batch — chapter 02 stratifies further). Note the time-of-day shape.
4. **Compute Little's Law.** `L = λ · W`. Compare to engine's running-batch gauge. Should agree to within 20% in steady state.
5. **Estimate ρ.** For decode-bound: `λ · W / (max_num_seqs · replicas)`. Target ≤ 0.70 for p99 survival; ≤ 0.85 for throughput optimisation with aggressive HPA (chapter 05).
6. **Check for HOL risk.** Plot TTFT p99 / TTFT p50 ratio over the last 24 h. Ratio > 5 suggests HOL or tail-of-prompt-length pathology.
7. **List the engine features that violate the model.** Continuous batching (always), prefix cache (if enabled), speculative decoding (if enabled), chunked prefill (if enabled), disaggregated P/D (if enabled), MultiLoRA (if enabled). Each entry is a reason to prefer measurement and simulation over derivation.

## Summary

- **An LLM request has two phases with different arithmetic.** Prefill is compute-bound and scales with prompt length. Decode is memory-bandwidth-bound and scales with completion length. Every capacity signal is interpretable only through this split.
- **Little's Law is the shortest useful queueing statement.** `L = λ · W` holds under weak assumptions, converts SLOs into batch-size budgets, and tells you when the fleet is queued before TTFT visibly regresses. It gives you means, not tails.
- **M/M/c is the first-order model.** It gets you an order-of-magnitude answer and makes the utilisation-vs-wait-time curve legible. The waiting-trap intuition — `ρ → 1` blows up — is the single most important shape to carry.
- **Head-of-line blocking is the common failure mode the first-order model misses.** Chunked prefill, disaggregated P/D, and admission-side bucketing are the engine's mitigations; the AI SRE notices HOL from TTFT p99/p50 ratio and routes to the engine peer.
- **Modern serving features violate the simple model.** Continuous batching, prefix caching, speculative decoding, chunked prefill, disaggregated P/D, and MultiLoRA all break the independence assumption of M/M/c. The upgrade path is calibrated simulation plus periodic load tests (chapter 06).
- **The worksheet is seven steps.** Name the phase profile, measure `W` and `λ`, compute Little's Law, estimate ρ, check for HOL risk, list violating features, and know when the first-order model is lying to you.

The next chapter takes the first-order model and extends it to a mixed-workload fleet — chat + RAG + tool-calling + batch, each with its own prompt / completion distribution, served from a single node pool.
