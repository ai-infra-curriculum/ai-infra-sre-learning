# vLLM Operator Fluency — Enough to Triage, Not to Author

## Why this chapter exists

[vLLM](https://docs.vllm.ai/) is the most widely deployed open-source inference engine in 2024–2026. If your platform serves any open-weight model, the probability that vLLM is somewhere in your stack is high enough that you have to be able to triage an outage against it at 03:00 without paging the inference-engine peer first. That is the goal of this chapter: *operator* fluency, not *authoring* fluency.

The source of truth is the [vLLM documentation](https://docs.vllm.ai/). This chapter curates the subset an on-call needs on the first read. Source-level depth — scheduler internals, PagedAttention block table implementation, kernel authoring, new feature development — is the peer's deliverable, covered in the `ai-infra-inference-engineer-learning` sibling curriculum.

## 1. The three vLLM concepts you need to recognize under pressure

### PagedAttention

vLLM's core memory primitive. The KV cache is chunked into fixed-size blocks (classically 16 tokens per block) and managed by a block manager analogous to an OS page table. Conceptually: each sequence's KV footprint is a list of block IDs; blocks can be shared across sequences; the memory is not contiguous.

**Why an operator cares.** When the engine reports it is out of KV cache, the *blocks-free* metric is the one to watch, not raw GPU memory. OOMs can happen with GPU memory nominally under budget because the KV pool is pre-allocated to a fixed fraction and then internally exhausted.

### Continuous batching

vLLM batches at the iteration level, not the request level. Each decode step, the scheduler picks the set of sequences to advance based on available KV blocks and the waiting queue. New requests can enter a batch mid-generation; completed sequences leave immediately.

**Why an operator cares.** This is what makes vLLM's throughput high and what makes its queue dynamics opaque. "Why is latency increasing?" often resolves to "the scheduler is admitting more concurrent sequences than fit comfortably, so each one's decode step is competing for the same KV pool."

### Prefix caching

If two requests share a prefix (system prompt, few-shot examples, long RAG context), vLLM can reuse the KV blocks for the shared prefix across requests — the prefix is computed once and its KV blocks are referenced by many sequences. Controlled by `--enable-prefix-caching`.

**Why an operator cares.** Prefix cache hit rate is a *huge* latency and cost lever. A system with a 90% prefix hit rate serves at much better TTFT than one with 10%. An unexpected TTFT regression often decomposes to a prefix-cache miss regression (flushed KV, changed system prompt, admission eviction) — covered in `mod-105`.

## 2. Launch flags — the ones that come up in triage

The authoritative list is in [`vllm serve` --help](https://docs.vllm.ai/en/latest/serving/engine_args.html). The shortlist for on-call:

### Parallelism

| Flag | Purpose |
|---|---|
| `--tensor-parallel-size N` (`-tp N`) | Shards the model across N GPUs on one node. Required for models larger than a single GPU's memory. |
| `--pipeline-parallel-size N` (`-pp N`) | Splits layers across pipeline stages. Needed for multi-node or for very large models. |
| `--data-parallel-size N` (`-dp N`) | Replicates the model N times for data parallelism. |
| `--distributed-executor-backend {ray,mp,...}` | How the parallel workers are coordinated. |

**Triage hook.** Mismatched `-tp` and GPU topology (e.g., `-tp 4` on a node with 3 healthy GPUs after one fails) is a common failure-to-start symptom.

### Memory and KV cache

| Flag | Purpose |
|---|---|
| `--gpu-memory-utilization F` | Fraction of GPU memory pre-allocated to the KV pool + weights (default ~0.9). |
| `--max-model-len N` | Maximum sequence length (prompt + generation). Bounds the KV footprint per sequence. |
| `--max-num-seqs N` | Maximum concurrent sequences in a batch. Admission limit. |
| `--max-num-batched-tokens N` | Maximum total tokens processed per iteration. Sets the prefill chunk ceiling. |
| `--swap-space G` | Gigabytes of CPU RAM to use as KV swap when GPU KV is full. |
| `--kv-cache-dtype {auto,fp8,fp8_e4m3,fp8_e5m2}` | Quantize KV cache to save memory. |

**Triage hook.** `--gpu-memory-utilization` set too high leaves no slack for activations during long prefills → OOM at request time; set too low wastes KV capacity. The classic first-week-of-production mis-setting.

### Prefix caching and chunked prefill

| Flag | Purpose |
|---|---|
| `--enable-prefix-caching` | Turn on prefix-cache reuse. |
| `--enable-chunked-prefill` | Allow prefills to be split across iterations and interleaved with decode. |

**Prefix caching triage hook.** If TTFT regresses suddenly, check:

1. Prefix cache hit rate on the vLLM metrics endpoint.
2. Whether a change to the system prompt made upstream invalidated the cache.
3. Whether admission-control evictions (triggered by a load spike) flushed warm blocks.

**Chunked prefill triage hook.** Chunked prefill is what prevents a long-prompt request from monopolizing the batch and starving decode. If ITL (inter-token latency) is spiking under long-context load, confirm chunked prefill is enabled; the knob is `--max-num-batched-tokens` (tunes the chunk ceiling).

### Speculative decoding

| Flag (modern vLLM) | Purpose |
|---|---|
| `--speculative-config` (JSON) | Full speculative-decoding config: draft model, number of speculative tokens, verification policy. |
| `--num-speculative-tokens N` | Number of draft tokens proposed per decode step. |

Older vLLM releases exposed `--speculative-model` and related flags as top-level args; modern releases consolidate into `--speculative-config`. Check the release-notes page for the version in your fleet.

**Triage hook.** Speculative decoding wins throughput when the draft model's acceptance rate is high (typically > 60%). Under drift — model updates, workload drift — acceptance can drop, and speculative decoding becomes a net loss. Monitor acceptance rate; disable the feature as a first step if TPOT regresses after a model update.

<!-- needs-research: confirm the exact flag names and deprecation status for speculative decoding across vLLM 0.5.x, 0.6.x, and 0.7.x; cite the specific docs page URL for the currently-canonical flag. -->

### Quantization

| Flag | Purpose |
|---|---|
| `--dtype {auto,half,bfloat16,float16,float32}` | Weights dtype. |
| `--quantization {awq,gptq,fp8,...}` | Weight quantization scheme at load time. |

**Triage hook.** Quantization-related failures usually surface as model-loads-but-outputs-are-garbage (bad weight parsing) or OOM-at-load (wrong scheme for the checkpoint). A production outage caused by quantization is usually a deploy-pipeline issue, not an on-call issue.

### Observability and debugging

| Flag | Purpose |
|---|---|
| `--disable-log-stats` / default-on | vLLM emits iteration-level stats to a logger. Keep stats on in production. |
| `VLLM_LOGGING_LEVEL=DEBUG` (env) | Verbose logging for triage. |

The vLLM OpenAI-compatible server exposes a `/metrics` Prometheus endpoint; wire it into your dashboard (`mod-104`).

## 3. The on-call vLLM triage flow

A triage runbook you should be able to walk without thinking.

### Step 1 — Is it classical infra or vLLM?

Check the four golden signals on the engine pod:

- GPU utilization / memory / SM clock — DCGM.
- Request rate and error rate — ingress / gateway.
- vLLM `num_running`, `num_waiting`, `gpu_cache_usage_perc` from `/metrics`.

If GPU util is high *and* `gpu_cache_usage_perc` is near 1.0, you have an admission / KV pressure problem. If GPU util is low *and* queue is backing up, you have a scheduler problem (or upstream auth / rate-limit backup). If GPU is crashing (XID codes, DCGM red), it is a hardware problem — escalate to the GPU peer.

### Step 2 — Is it a latency or a quality regression?

Pull the SLO stack. TTFT regression is almost always a prefix-cache issue, an admission issue, or a prefill saturation. TPOT regression is an iteration-time / scheduler issue. Quality regression is not a vLLM problem; it is upstream (prompt change, retrieval, model artifact) — route to `mod-105` triage.

### Step 3 — What changed?

- Recent engine deploy / version upgrade?
- Recent launch-flag change?
- Recent model swap?
- Recent prefix-cache policy change?

The audit log of launch flags and engine versions across the fleet is a reliability artifact; keep it in the same repo as your runbooks.

### Step 4 — Knob levers available without paging the peer

- Reduce `--max-num-seqs` to shed load if the KV pool is thrashing.
- Reduce `--max-num-batched-tokens` to reduce prefill head-of-line blocking.
- Toggle `--enable-chunked-prefill` on if off, to interleave prefill and decode.
- Flip `--enable-prefix-caching` off *as an experiment*, not a fix — if TTFT improves with caching off, the cache was corrupted or thrashing.
- Disable speculative decoding if acceptance rate is low.
- Reduce `--gpu-memory-utilization` if OOM-at-request is observed.

Each of these is a reversible operational change. If none restore SLO within the policy window, escalate to the inference-engine peer with the triage artifact and let them decide whether source-level work is needed.

## 4. The authoring boundary — what you do not do

You do not:

- Author a new scheduler policy.
- Add a kernel path for a new attention variant.
- Modify the PagedAttention block allocator.
- Patch the speculative-decoding verification loop.
- Add a new quantization backend.

All of the above live with the `ai-infra-inference-engineer` peer at level 35. When you have narrowed the failure to one of these, you file a ticket against the engine — with the triage artifact from step 1–3 above as attachments — and either (a) wait for the peer's fix with a documented workaround, or (b) roll the fleet back to the prior engine version if the workaround is insufficient and the policy allows the rollback.

## Summary

- **Three concepts to recognize.** PagedAttention (KV block manager), continuous batching (iteration-level scheduler), prefix caching (cross-request KV reuse).
- **Launch flags that come up in triage.** Parallelism (`-tp`, `-pp`, `-dp`), memory / KV (`--gpu-memory-utilization`, `--max-num-seqs`, `--max-num-batched-tokens`, `--kv-cache-dtype`, `--swap-space`), prefix / prefill (`--enable-prefix-caching`, `--enable-chunked-prefill`), speculative decoding (`--speculative-config`), quantization (`--dtype`, `--quantization`).
- **Triage flow.** Classical infra vs. engine → latency vs. quality → what changed → reversible knob levers → escalate with the artifact.
- **Authoring boundary.** You tune the knobs; the inference-engine peer owns the source.

The next chapter does the same walk for SGLang.
