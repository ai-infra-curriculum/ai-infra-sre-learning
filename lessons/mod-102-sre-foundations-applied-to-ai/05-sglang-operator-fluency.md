# SGLang Operator Fluency — RadixAttention and the DP-Attention Knobs

## Why this chapter exists

[SGLang](https://docs.sglang.ai/) is the second engine you should be able to triage without paging the inference-engine peer. Its design differs from vLLM in two ways that matter for operators: the KV cache is organized as a **radix tree** (RadixAttention) rather than a flat block pool, which makes prefix-sharing among structurally-similar prompts much more efficient; and it supports **DP-attention** — a data-parallel-attention mode that changes how tensor-parallel and data-parallel interact at the attention layer — which comes up as a knob on multi-GPU deployments.

The source of truth is the [SGLang documentation](https://docs.sglang.ai/). This chapter is the subset an on-call needs on first read. Source-level depth belongs to `ai-infra-inference-engineer`.

## 1. The two SGLang concepts you need to recognize

### RadixAttention

SGLang organizes the KV cache as a **radix tree** keyed by the token sequence. Multiple requests that share a prefix (system prompt, tool-use prelude, RAG header) share the same tree nodes; the KV blocks under those nodes are physically reused without copying. Eviction is LRU with reference-counting.

**Why an operator cares.** On workloads with high structural overlap — chat with a system prompt, tool-calling agents, few-shot RAG — RadixAttention produces much higher prefix hit rates than a flat prefix cache, with smaller memory. On workloads with low overlap (one-shot summarization of distinct documents), the tree is shallow and the win vanishes. The *hit rate* metric is the first signal to pull when debugging TTFT.

### DP-attention

On a multi-GPU deployment, the classical choice is TP (tensor-parallel: shard a single replica across GPUs) vs. DP (data-parallel: run multiple replicas, each handling a subset of requests). SGLang supports a hybrid where the **attention layer is data-parallel** while the rest of the model is tensor-parallel. The practical effect: the attention computation (which is memory-bandwidth bound on decode) is not serialized across the TP group, improving throughput on decode-heavy workloads.

**Why an operator cares.** DP-attention is a launch-time choice that materially changes the throughput / latency profile of the deployment. If your deployment is decode-heavy (long generations, agent loops) and is running without DP-attention, enabling it is a knob change — not a code change — with measurable impact. Conversely, enabling it on a prefill-heavy workload may not help and may complicate scheduling. The flag spelling has evolved: current SGLang uses `--attention-data-parallel-size` (alias `--attn-dp-size`), replacing the older `--enable-dp-attention` toggle — see the [SGLang server-arguments docs](https://docs.sglang.ai/advanced_features/server_arguments.html).

## 2. Launch flags that come up in triage

The authoritative reference is the SGLang launch-server documentation. The shortlist:

### Parallelism

| Flag | Purpose |
|---|---|
| `--tp-size N` | Tensor-parallel size. |
| `--dp-size N` | Data-parallel size. Replicas. |
| `--enable-dp-attention` | Enable data-parallel-attention as described above. |
| `--nnodes` / `--node-rank` / `--dist-init-addr` | Multi-node distributed init. |

### Memory and KV

| Flag | Purpose |
|---|---|
| `--mem-fraction-static F` | Fraction of GPU memory used for weights + static allocations. The rest is KV + activations. Analogous to vLLM's `--gpu-memory-utilization`. |
| `--max-total-tokens N` | Total token budget across the KV pool. Bounds concurrency. |
| `--max-prefill-tokens N` | Cap on prefill tokens per iteration. Prevents prefill from starving decode. |
| `--context-length N` | Max context length per request. |
| `--chunked-prefill-size N` | Chunk size for chunked prefill (interleaves prefill and decode). |

### RadixAttention

| Flag | Purpose |
|---|---|
| `--disable-radix-cache` | Turn RadixAttention off. Useful as a diagnostic — if TTFT is inconsistent and you suspect cache thrash, toggling off narrows the hypothesis. Do not leave off in production. |
| `--schedule-policy {lpm,fcfs,...}` | Request-scheduling policy. `lpm` (longest-prefix-match) exploits RadixAttention; `fcfs` is strict first-come-first-serve. Changing this is a performance knob, not a correctness change. |

### Quantization and dtype

| Flag | Purpose |
|---|---|
| `--dtype` | Weights dtype (bfloat16, float16, float32, auto). |
| `--kv-cache-dtype` | KV cache quantization (fp8_e4m3, fp8_e5m2). |
| `--quantization` | Model quantization (awq, gptq, fp8, etc.). |

### Speculative decoding

SGLang supports speculative decoding via EAGLE and similar draft-model paths. The current canonical flags are `--speculative-algorithm`, `--speculative-draft-model-path`, and `--speculative-num-steps` (with additional knobs such as `--speculative-eagle-topk` and `--speculative-num-draft-tokens`) — see the [SGLang server-arguments docs](https://docs.sglang.ai/advanced_features/server_arguments.html). Treat it operationally like vLLM: it is a throughput optimization that *can* regress TPOT when acceptance rate drops; monitor the rate.

### Observability

SGLang exposes a `/metrics` endpoint analogous to vLLM. Wire it the same way (`mod-104`). The RadixAttention cache-hit-rate signal is the most SGLang-specific metric on the dashboard.

## 3. The SGLang triage flow

Mirrors the vLLM flow in chapter 04; the differences are below.

### Step 1 — Classical infra vs. engine

Same as vLLM. DCGM for the GPU, ingress for traffic, `/metrics` for engine internals.

### Step 2 — Latency vs. quality

TTFT regression → pull the RadixAttention hit-rate metric first. A drop from a historical baseline (say 85% → 30%) is the signal that the prefix-sharing structure of the traffic changed or the cache was evicted.

TPOT regression on a decode-heavy workload → check whether DP-attention is enabled; on multi-GPU deployments, enabling it is a frequent throughput win.

Prefill saturation symptoms (ITL spikes when long prompts arrive) → check `--chunked-prefill-size` and `--max-prefill-tokens`. Tune down to give decode more scheduling slack.

### Step 3 — What changed

The audit list is the same as vLLM — engine version, launch flags, model swap, cache policy. SGLang's release cadence is rapid; schema and flag names move between versions. Pin the version in the audit log.

### Step 4 — Knob levers

- Lower `--mem-fraction-static` if OOM pressure is on activations.
- Lower `--max-total-tokens` / `--max-prefill-tokens` to shed load.
- Enable `--enable-dp-attention` on decode-heavy multi-GPU deployments as a throughput experiment.
- Toggle `--schedule-policy` between `lpm` and `fcfs` to isolate prefix-related anomalies.
- Toggle `--disable-radix-cache` as a *diagnostic*, not a fix.
- Disable speculative decoding if acceptance rate is low.

## 4. SGLang vs. vLLM — the operator's mental model

You will often triage hybrid fleets where one product is on vLLM and another on SGLang. The differences that matter at the pager:

| Dimension | vLLM | SGLang |
|---|---|---|
| KV layout | Paged blocks, flat pool | Radix tree with reference-counting |
| Prefix caching | `--enable-prefix-caching`, flat-prefix match | RadixAttention, structural prefix match |
| Scheduling | Continuous-batching scheduler, FCFS-like | LPM-aware scheduler on `lpm` policy |
| Memory knob | `--gpu-memory-utilization` | `--mem-fraction-static` |
| Concurrency knob | `--max-num-seqs`, `--max-num-batched-tokens` | `--max-total-tokens`, `--max-prefill-tokens` |
| Multi-GPU attention | TP across GPUs | TP + optional DP-attention |

If an engineer tells you "the prefix cache isn't helping on service X," your next question is which engine X runs; the diagnostic and the knob are different.

## 5. The authoring boundary

Same as vLLM. You tune knobs; you file tickets upstream; you do not author the RadixAttention eviction policy, the DP-attention kernel path, or the scheduler. If a bug in any of those is the root cause, the inference-engine peer owns the fix.

## Summary

- **Two concepts.** RadixAttention (radix-tree KV cache with structural prefix sharing) and DP-attention (data-parallel-attention mode for multi-GPU decode-heavy workloads).
- **Launch flags.** Parallelism (`--tp-size`, `--dp-size`, `--enable-dp-attention`), memory / KV (`--mem-fraction-static`, `--max-total-tokens`, `--max-prefill-tokens`, `--chunked-prefill-size`), RadixAttention (`--disable-radix-cache`, `--schedule-policy`), quantization and speculative decoding analogous to vLLM.
- **Triage flow.** Same shape as vLLM; the SGLang-specific metric is the RadixAttention hit rate, and the SGLang-specific knob is DP-attention.
- **Authoring boundary.** Tune knobs, file tickets, do not patch engine source.

The next chapter walks TensorRT-LLM + Triton Inference Server + NVIDIA Dynamo — the NVIDIA-backed serving stack for workloads where the extra engine-build complexity is justified by the latency and throughput gains.
