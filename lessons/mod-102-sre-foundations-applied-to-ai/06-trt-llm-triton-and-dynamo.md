# TensorRT-LLM, Triton Inference Server, and NVIDIA Dynamo

## Why this chapter exists

The NVIDIA-backed open-source serving stack — **TensorRT-LLM** for engine-level optimization, **Triton Inference Server** for the serving process, and **NVIDIA Dynamo** for disaggregated serving — is the third family of inference engines you must be able to triage. It is structurally different from vLLM and SGLang: TensorRT-LLM engines are *compiled artifacts*, built ahead of time against a specific GPU architecture and a specific set of shapes; Triton is a general inference server that hosts TRT-LLM engines via a backend; Dynamo is a disaggregated-serving orchestrator that separates prefill and decode workers.

The operator's job here is to understand the **engine-build workflow** well enough to triage a serving outage whose root cause is upstream of the running server (bad build config, incompatible checkpoint, shape mismatch), and to understand the **Triton wiring** well enough to debug a config.pbtxt that is producing the wrong error. Source-level engine authoring stays with the inference-engine peer.

Documentation:

- [TensorRT-LLM](https://nvidia.github.io/TensorRT-LLM/)
- [Triton Inference Server](https://docs.nvidia.com/deeplearning/triton-inference-server/)
- [NVIDIA Dynamo](https://docs.nvidia.com/dynamo/)

## 1. TensorRT-LLM — the engine-build workflow

### The three-stage workflow

Unlike vLLM or SGLang, a TensorRT-LLM deployment is not "launch the engine against a HuggingFace checkpoint." It is a three-stage pipeline:

1. **Checkpoint conversion.** Convert the model's HuggingFace / native checkpoint into TensorRT-LLM's canonical format (a directory of weights + a config describing the model architecture). Each model family has a converter script in the TensorRT-LLM examples tree.

2. **Engine build.** Run `trtllm-build` against the converted checkpoint to produce a **compiled engine** — a `.engine` binary tuned for a specific GPU architecture, dtype, batch-size range, and sequence-length range. The build is a non-trivial step (minutes to hours depending on model size) and produces an artifact that must be rebuilt when any of the above changes.

3. **Serving.** Host the engine behind Triton Inference Server using the TensorRT-LLM backend, or (for development) use `trtllm-serve` for a direct launch. Triton exposes the HTTP / gRPC endpoint and handles batching, scheduling, and queueing.

**Why an operator cares.** An engine built for max-batch=32, max-input-len=4096 cannot be used for a request with batch=64 or input-len=8192. The failure is at request time, not at engine load. "The engine won't serve this request" is a *build* issue, not a *runtime* issue, and the fix is a rebuild — not a knob change.

### Key `trtllm-build` arguments to recognize

| Flag | Purpose |
|---|---|
| `--max_batch_size N` | Maximum concurrent batch size the engine supports. |
| `--max_input_len N` | Maximum prompt length. |
| `--max_seq_len N` | Maximum total sequence length (prompt + generation). |
| `--gemm_plugin {auto,bfloat16,float16,fp8}` | Which plugin for the GEMM kernels. |
| `--gpt_attention_plugin ...` | Attention kernel plugin. |
| `--paged_kv_cache enable` | Use paged KV cache (analogous to vLLM's PagedAttention). |
| `--use_paged_context_fmha enable` | Use paged fused multi-head attention for prefill. |
| `--kv_cache_type {paged,continuous}` | KV cache layout. |
| `--remove_input_padding enable` | Pack variable-length inputs without padding. |

**Flag naming has evolved across TensorRT-LLM versions.** Pin your version and point the runbook at the correct docs page — the current reference is the [`trtllm-build` docs](https://nvidia.github.io/TensorRT-LLM/commands/trtllm-build.html). Note that `--paged_kv_cache` has been superseded by `--kv_cache_type {paged,continuous,disabled}`; treat the old flag as a compatibility alias on the way out.

### Build-stage failure classes

Three common failure modes you will see:

1. **Shape overflow at request time.** Request exceeds `max_batch_size` or `max_input_len` → engine rejects. Fix: rebuild with wider shapes, *or* clamp the incoming request range in the gateway. A good operational default: set the serving-side admission limit *inside* the engine's declared shape range, so admission errors are the gateway's problem, not the engine's.

2. **Checkpoint-engine mismatch.** Engine was built against checkpoint rev A; checkpoint rev B was promoted; engine load fails with a weight-shape mismatch. Fix: rebuild the engine against the current checkpoint; keep engine builds versioned alongside the checkpoint they target.

3. **GPU architecture mismatch.** Engine built on H100, deployed on A100 — fails to load. Fix: build matrix per GPU architecture in use; the deploy pipeline must route the right engine to the right pool.

The engine-build workflow is itself a reliability surface. Treat it like any other build pipeline: versioned artifacts, reproducible builds, build-provenance metadata (which checkpoint, which flags, which CUDA + TRT version produced this engine), rollback.

## 2. Triton Inference Server — the wiring layer

[Triton Inference Server](https://docs.nvidia.com/deeplearning/triton-inference-server/) is a general-purpose inference server; it hosts models via backends (TensorRT-LLM backend, Python backend, PyTorch backend, ONNX Runtime backend, etc.) and provides the HTTP / gRPC layer, batching, scheduling, metrics, and health endpoints.

### The model repository layout

Triton reads a **model repository** — a filesystem path with one subdirectory per model. Each model directory contains:

- `config.pbtxt` — the model-config file (protobuf text format).
- `1/` (or `2/`, `3/`) — numbered version directories containing the actual model artifact (the TRT-LLM engine, the ONNX file, the Python backend script, etc.).

### `config.pbtxt` — the knobs that matter

For a TRT-LLM-backed model, the config file names:

- The **backend** (`backend: "tensorrtllm"` or similar).
- The **input / output tensor spec** (names, dtypes, shapes).
- The **instance group** — how many engine instances to run per GPU and how to distribute them across GPUs.
- **Dynamic batching** — whether Triton batches at the request level in addition to the engine's internal batching, and the max-batch and queue-delay parameters.
- The **model transaction policy** (decoupled for streaming outputs, coupled for single-response).

**Why an operator cares.** Wrong `instance_group` (asking for more instances than fit in GPU memory) → server fails to start. Wrong `max_batch_size` in `config.pbtxt` vs. the engine's build-time `max_batch_size` → request-time errors. Wrong transaction policy (coupled when the client expects streaming) → users see one big blob instead of a stream.

### Triton as an operator surface

Triton exposes the same operator-friendly surfaces as vLLM and SGLang:

- `/metrics` Prometheus endpoint (comprehensive — per-model latency histograms, batch-size distributions, backend-specific signals including TRT-LLM KV-cache usage).
- `/v2/health/ready` and `/v2/health/live` for Kubernetes probes.
- `tritonserver --log-verbose=1` for triage logging.

### Triton triage flow

Different from vLLM / SGLang because Triton is a layer above the engine. The triage split:

- **If the symptom is latency or throughput** → the knob is in the engine (TRT-LLM build) or the Triton dynamic-batching config. Both are legitimate operator surfaces.
- **If the symptom is "requests are rejected"** → check `config.pbtxt` shape declarations and engine shape range.
- **If the symptom is "model won't load"** → instance-group fits in GPU, engine file present, engine architecture matches the GPU.
- **If the symptom is streaming behaviour** → transaction policy and backend output decoupling.

## 3. NVIDIA Dynamo — disaggregated serving

[NVIDIA Dynamo](https://docs.nvidia.com/dynamo/) is a newer entry (2024–2025) in the serving stack. Its central idea is **disaggregated serving**: separating the prefill phase and the decode phase onto *different pools of GPUs* rather than running both on the same workers.

### Why disaggregate prefill and decode

Prefill and decode have fundamentally different compute profiles.

- **Prefill** is FLOP-dominated. Processing a 4k-token prompt is one large matrix multiplication per layer; it is compute-bound. A GPU with high peak FLOPs wins.
- **Decode** is memory-bandwidth dominated. Generating each successive token loads the KV cache and the weights; the arithmetic intensity is low. A GPU with high memory bandwidth and large HBM wins.

When both phases run on the same GPU under continuous batching, the scheduler has to interleave them — which is why vLLM and SGLang need chunked prefill and tunable `max-batched-tokens`. The interleaving leaves throughput on the table because the same hardware is being asked to be good at two different things.

**Disaggregated serving** addresses this by running prefill on one pool of workers, decode on another, and transferring the computed KV cache between them. The pools can be sized independently (more prefill workers when traffic is prompt-heavy, more decode workers when traffic is generation-heavy) and can use different GPU shapes (compute-optimized for prefill, memory-optimized for decode).

### Dynamo's operator-facing surfaces

- **Prefill workers.** A pool configured for prompt processing.
- **Decode workers.** A pool configured for generation.
- **KV transfer layer.** The mechanism that moves the KV cache from prefill to decode workers. Latency and bandwidth of this transfer are observable and tunable.
- **Smart routing / placement.** The component that decides which prefill worker handles the prompt and which decode worker handles the generation.

### Why an operator cares

Three things to recognize:

1. **The architecture has one more hop.** Prefill to decode KV transfer adds latency on top of the prefill-only or decode-only path. A disaggregated deployment's TTFT budget is different from a co-located deployment's. Factor the transfer into your SLO calculations.

2. **Pool sizing is now a 2D problem.** Capacity planning (`mod-106`) has to account for prefill pool and decode pool separately. A workload shift from short prompts to long prompts raises the prefill pool pressure without raising decode; you may need to resize one pool and not the other.

3. **Failure domains split.** A prefill-worker fault affects only prefill capacity; a decode-worker fault affects only ongoing generations. Your chaos experiments (`mod-109`) should exercise both independently.

### When disaggregated serving is the right choice

This is a design-time decision, not an operator knob:

- **Decide to disaggregate when** the workload is large, the mix of prefill-heavy and decode-heavy requests is bimodal, and the throughput gain justifies the operational complexity.
- **Decide to co-locate (vLLM / SGLang / TRT-LLM on Triton)** when the workload is modest, the mix is uniform, or operational simplicity is more valuable than peak throughput.

NVIDIA's published numbers give a feel for the magnitude: Dynamo with disaggregated serving reports up to a ~2× throughput gain on Llama-70B on Hopper GPUs (vLLM backend, FP8, ISL/OSL 3K/50) and up to ~30× more requests served on DeepSeek-R1 on GB200 NVL72 (TensorRT-LLM backend, FP4, ISL/OSL 32K/8K) vs. co-located inflight batching — see the [NVIDIA Dynamo launch blog](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models). The gain is highly workload- and hardware-dependent; treat these as upper-end motivational figures, not deployment forecasts.

## 4. The operator cheatsheet across all three engine families

Compressed form:

| Dimension | vLLM | SGLang | TRT-LLM + Triton | Dynamo |
|---|---|---|---|---|
| Engine form | Python server against HF checkpoint | Python server against HF checkpoint | Compiled engine artifact + Triton host | Disaggregated prefill + decode workers |
| Build step | None | None | `trtllm-build` ahead of time | Varies by backend engine |
| Prefix caching | `--enable-prefix-caching` | RadixAttention (on by default) | Via TRT-LLM paged KV + plugin flags | Inherited from backend engine |
| Prefill / decode split | Interleaved with chunked prefill | Interleaved with chunked prefill | Interleaved with chunked prefill | **Physically separated pools** |
| Metric endpoint | `/metrics` | `/metrics` | Triton `/metrics` (comprehensive) | Varies |
| Primary triage knob | `--max-num-seqs`, `--max-num-batched-tokens` | `--max-total-tokens`, `--max-prefill-tokens` | Triton `instance_group` + engine build shapes | Pool sizing + routing policy |

The same vocabulary (SLIs, burn rates, triage flow) applies across all four. The knobs change; the methodology does not.

## 5. The authoring boundary

Same rule as chapters 04 and 05. The operator:

- Reads and modifies `config.pbtxt`.
- Requests engine rebuilds via the build pipeline (or kicks one off, if the team owns the pipeline).
- Tunes instance_group, max-batch, dynamic-batching knobs.
- Rebalances Dynamo pool sizes.
- Switches between engine backends via deploy config.

The inference-engine peer:

- Authors TRT-LLM plugins.
- Writes new Triton backends.
- Modifies Dynamo's routing or KV-transfer internals.
- Patches CUDA kernels behind any of the above.

A good signal of the correct boundary: the operator's change is a config edit or a rebuild; the peer's change is a C++ / CUDA / Python-source commit.

## Summary

- **TensorRT-LLM engine build is a reliability surface** — compiled engines pinned to GPU architecture, batch-size range, sequence-length range. Shape overflow, checkpoint mismatch, and GPU-architecture mismatch are the three canonical failure classes. Version the build pipeline.
- **Triton Inference Server is the wiring layer** — model repository with `config.pbtxt`, instance_group, dynamic batching, transaction policy. The operator reads and edits these; the peer writes new backends.
- **NVIDIA Dynamo is the disaggregated-serving option** — prefill pool, decode pool, KV transfer, smart routing. Chosen at design time for bimodal large workloads; adds an architectural hop and splits failure domains.
- **The methodology is unchanged across all three engine families.** Only the knob names and build mechanics differ.

The next chapter closes the loop on the peer boundary: *given* engine-knob fluency across vLLM, SGLang, and the TRT-LLM / Triton / Dynamo family, what is the explicit line between you and the inference-engine peer?
