# Exercise 04 — vLLM / SGLang / TRT-LLM Engine-Knob Cheatsheet

**Estimated effort:** 3 hours · **Output:** one Markdown cheatsheet file (~2000 words) + a triage runbook appendix

## Objective

Produce a **one-page-per-engine cheatsheet** that the primary on-call pulls up at 03:00 when a page says the engine is in trouble. Cover vLLM, SGLang, and TensorRT-LLM + Triton; add NVIDIA Dynamo if your workload uses disaggregated serving. Then author a **triage runbook** that maps symptoms to the right cheatsheet's knobs.

The point of this exercise is **recall under pressure**. If the artifact is organized in a way that requires reading five pages to find the right knob, it has failed its purpose. One page per engine, symptom-indexed.

## Prerequisites

- Chapters 04, 05, and 06 of this module.
- Exercises 01–03 (same workload context carries through; the SLOs from exercise 01 are what the knobs protect).
- Access to the official documentation for each engine. Links in `resources.md`.

## Problem statement

You are standing up the engine-knob runbook for your platform. The platform runs some combination of vLLM, SGLang, TensorRT-LLM + Triton, and NVIDIA Dynamo. The product organization pages your rotation when latency or availability SLOs burn; you need a triage artifact that resolves "which engine's knob do I touch, and what does the knob do" without a 30-minute source-code dive.

Author one cheatsheet per engine family you run, plus one top-level triage runbook that routes symptoms to the right cheatsheet.

## Requirements

### Artifact 1 — Per-engine cheatsheets

For each engine family (minimum three: vLLM, SGLang, TRT-LLM + Triton; add Dynamo if applicable), produce a cheatsheet with these sections:

#### Header

- Engine name, pinned version in your platform, commit SHA if available.
- Link to the pinned docs page.
- Link to the owning peer rotation / channel (for escalations per chapter 07).

#### Launch-flag reference (operator-focused)

A table with four columns: *flag, purpose, triage hook (when does this matter?), reversible?*

Minimum rows from the chapter for each engine:

- **vLLM** — `--tensor-parallel-size`, `--pipeline-parallel-size`, `--gpu-memory-utilization`, `--max-model-len`, `--max-num-seqs`, `--max-num-batched-tokens`, `--enable-prefix-caching`, `--enable-chunked-prefill`, `--kv-cache-dtype`, `--swap-space`, speculative-decoding config, `--dtype`, `--quantization`.
- **SGLang** — `--tp-size`, `--dp-size`, `--enable-dp-attention`, `--mem-fraction-static`, `--max-total-tokens`, `--max-prefill-tokens`, `--chunked-prefill-size`, `--schedule-policy`, `--disable-radix-cache`, `--dtype`, `--kv-cache-dtype`, `--quantization`.
- **TRT-LLM + Triton** — `trtllm-build` arguments (`--max_batch_size`, `--max_input_len`, `--max_seq_len`, `--gemm_plugin`, `--paged_kv_cache`, `--use_paged_context_fmha`, `--kv_cache_type`, `--remove_input_padding`) + Triton `config.pbtxt` fields (backend, input / output spec, instance_group, dynamic batching, transaction policy).
- **Dynamo (if applicable)** — prefill pool size, decode pool size, routing policy, KV-transfer tuning knobs.

#### Metric-signal reference

A short table of the engine-specific metric names from each engine's `/metrics` endpoint that are first-looked under pressure. E.g., for vLLM: `gpu_cache_usage_perc`, `num_running`, `num_waiting`, prefix-cache hit rate. For SGLang: RadixAttention hit rate, decode-step time.

#### Symptom → knob index

A short table mapping the three or four most common symptoms (TTFT regression, TPOT regression, OOM, admission-rejection errors) to the knob to adjust first. Structured like:

| Symptom | First knob to adjust | Second knob | What to check on the dashboard |
|---|---|---|---|
| TTFT regression | Prefix-cache hit rate; toggle `--enable-prefix-caching` | Lower `--max-num-seqs` if KV pressure | Hit rate and `gpu_cache_usage_perc` |

#### Hand-off boundary

A two-line summary of the chapter-07 contract: what you (operator) can change inside the engine's config vs. when to open a ticket to the engine peer.

### Artifact 2 — Top-level triage runbook

A separate section (or separate file) that routes an incoming page to the right cheatsheet.

Required rows:

- **Symptom: 5xx spike on `/v1/chat/completions` from engine pool.**
- **Symptom: TTFT p99 breaching SLO.**
- **Symptom: TPOT p99 breaching SLO.**
- **Symptom: ITL spiking on long-prompt requests.**
- **Symptom: engine pod OOM.**
- **Symptom: engine pod CrashLoop / failed-to-start.**
- **Symptom: KV-cache saturation (engine-specific signal).**
- **Symptom: quality SLO burning with no latency / availability signal.** (Hint: this is *not* an engine problem; route to `mod-105` triage, not to the cheatsheet.)

For each row:

- Which engine is affected (which pool?).
- Which cheatsheet section applies.
- Reversible first-step knob.
- Escalation condition (when to open a ticket).

## Starter guidance

- The authoritative reference for each engine is its own docs. Do not paraphrase from memory; verify the flag name and default against the pinned version's docs. Flag naming changes across minor versions; a cheatsheet that cites deprecated flags is worse than no cheatsheet.
- The triage runbook is the *first-read* artifact. The cheatsheets are the *deep-dive* artifact. Make sure the runbook can be read in two minutes.
- The symptom → knob index inside each cheatsheet is the second-most-valuable piece of the exercise (after the triage runbook). Spend time there.
- When a symptom's resolution requires "read the engine source," that is the hand-off boundary. Mark the row explicitly: "Escalate to inference-engine peer after the reversible knobs are exhausted."

## Acceptance criteria

- One cheatsheet per engine family deployed on your platform (minimum three).
- Each cheatsheet has the five sections (header, launch-flag table, metric-signal table, symptom→knob index, hand-off boundary).
- Launch-flag table cites the flag names *as they exist in the pinned version of the engine docs* (not from memory).
- Triage runbook covers the eight symptom rows above.
- The quality-SLO-burning row correctly routes out of this cheatsheet (to `mod-105`).
- A primary on-call reading the triage runbook on their first page can find the right cheatsheet in under two minutes.

## Stretch goals

- **Workload-specific tuning notes.** For each engine, add a short section of notes on what tuning is right *for your workload* — e.g., "our workload is 80% long-prompt summarization, so `--enable-chunked-prefill` is on and `--max-num-batched-tokens` is tuned to 4096." This specialization makes the cheatsheet operable by someone who has not seen your workload before.
- **Version-diff appendix.** If your platform has moved between engine versions recently, add a one-line diff of flag changes that caused real operational surprises. This is the institutional memory the cheatsheet is otherwise lacking.
- **Dry-run with a real incident.** Walk the cheatsheet against a real post-mortem from your team (or a public one). Does the triage runbook route to the right cheatsheet? Does the symptom→knob index point at the knob that actually resolved the incident? Where it does not, revise.
- **Reverse-direction: what the engine peer hands off to you.** A short closing section with the chapter-07 reverse-direction cases (engine release rollout, hotfix rollout, engine configuration recommendation, capacity signal). This closes the loop on the peer contract.
