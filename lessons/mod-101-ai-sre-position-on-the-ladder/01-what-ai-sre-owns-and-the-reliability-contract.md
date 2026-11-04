# What AI SRE Owns — The Reliability Contract Around the AI Stack

## Why this chapter exists

Before anything else — latency SLOs, GPU telemetry, chaos drills, vendor fallback — you need a single sentence you can repeat under pressure that says what your team is for. SRE teams that cannot answer *"what do you own?"* in one breath end up owning everything the serving engineers and application teams do not want to own, which is a career-shortening mistake in both directions: the team drowns, and the real reliability work does not get done because nobody has time to do it.

This chapter is that sentence, unpacked.

## The one-sentence charter

> **AI SRE owns the operational reliability contract of production AI, ML, LLM, and agent systems.**

Four words in that sentence do the work. Each is deliberate.

- **Operational.** Running systems — not designing them from scratch, not training the model, not writing the application prompt. The surface that is live, taking traffic, and costing money right now.
- **Reliability.** Availability, latency, correctness-under-load, cost-per-successful-request, graceful degradation. The dimensions users and finance notice when they go bad.
- **Contract.** A written, enumerable, negotiable agreement — SLOs, error budgets, runbooks, on-call rotations, incident-response commitments. Not a vibe. Not "we try hard." A contract is what makes reliability work legible to the business.
- **AI / ML / LLM / agent systems.** The scope is the AI surface specifically. If your company also runs a non-AI CRUD app, that app's reliability is some other SRE team's problem. You cover the GPUs, the serving engines, the model-provider calls, the prompts-as-config, the retrieval pipelines, the agent orchestrators.

## What that unpacks into — the five owned surfaces

These are the five nouns a hiring manager will put in your job description, and they are the same five your own charter should list when you write it later in this module.

1. **SLO engineering for model-backed services.** Per-token latency SLOs (TTFT, TPOT, ITL), quality SLOs (hallucination rate, toxic-output rate, tool-call success, structured-output conformance), availability-under-streaming, cost-per-1M-tokens. The methodology is classical SRE; the SLIs are AI-specific. Covered depthwise in `mod-103`.

2. **Observability for LLM request paths and GPU workloads.** OpenTelemetry `gen_ai.*` semantic conventions on the request side; DCGM / Node Problem Detector / XID codes on the GPU side; LLM-trace platforms (Langfuse / Phoenix / LangSmith) and drift platforms (Arize / WhyLabs / Evidently) wired in as SRE signals, not product-analytics curiosities. Covered depthwise in `mod-104`.

3. **Incident response for non-deterministic failures.** Silent output drift, hallucination spike, toxic blast, prompt-cache cliff, retrieval-freshness regression, agent cascade, judge-model drift, provider-degraded mode. These classes do not exist in classical SRE; they do not look like a CPU-pegged request path or a failing health check. They are the reason this role exists. Covered in `mod-105` and `mod-108`.

4. **Capacity, cost, and vendor risk for AI workloads.** GPU fleet queueing-theoretic headroom, cross-generation fleet mix, HPA on inference-specific metrics, Karpenter GPU bin-packing, model-provider quota pools, fallback contracts to a second provider, SLA-vs-SLO reconciliation, status-page ingestion into the incident pipeline. Covered in `mod-106` and `mod-107`.

5. **Chaos, release, DR, and the cross-functional ritual.** Chaos experiments targeted at the failure classes above; progressive-delivery for model updates with SLO-based auto-rollback; DR / BCP / multi-region for stateful AI workloads; the SLO review ritual, build-vs-buy RFC posture, and cross-peer interface contracts that keep the program running. Covered in `mod-109` through `mod-114`.

## What AI SRE does *not* own

This list is at least as important as the one above, because every hour you spend on these is an hour you are not spending on the reliability contract.

- **The serving engine's source code.** `vLLM`, `SGLang`, `TensorRT-LLM`, `Dynamo` — you operate them, you file tickets against them, you tune their knobs. You do not author their scheduler loops or KV allocators. That is the `ai-infra-inference-engineer` peer at the same level.
- **CUDA / kernel-authoring depth.** If a kernel regresses or you need a new FlashAttention variant, you co-file a ticket; you do not open Nsight-Compute, read SASS, and write a replacement. That is the `ai-infra-performance` peer at the same level.
- **Adversarial defence.** Prompt-injection classifiers, PII-filter sidecars, model-extraction rate control, ML-BOM signing — all `ai-infra-security` peer at the same level. You own the *reliability knock-on* of their controls (latency budget consumed by a guardrail, cache impact of a PII filter) but not the controls themselves.
- **Model training and fine-tuning.** The upstream model artifact is handed to you with a resource profile; you do not decide what it learns. That is the ML Engineering family.
- **The product prompt and the agent design.** Prompt engineering, tool definitions, UX copy — AI Engineering family. You own the reliability of *delivering* their prompt, not the prompt itself.
- **The eval set.** Quality evaluation methodology, judge-vs-human calibration, eval-set construction — the `ai-eval-engineer` and `model-evaluation-engineer` peers at level 30. You consume their signal and wire it into release gates; you do not author the gate's correctness criterion.

If any of these land in your queue, the correct move is **route, do not absorb**. The reliability charter you author in `exercise-03` is the artifact that does this routing in writing, so the routing decision does not get re-litigated every time.

## Why the role exists at level 35 specifically

Level 35 in this org's ladder is an AI-reliability *specialist* — a mid-to-staff engineer band with AI-specific scope, not a generalist SRE band and not yet a team-lead / architect scope.

The role could, in principle, be done by a very experienced general SRE. It usually is not, because the failure modes above require enough AI-specific literacy (what a KV cache is, what TTFT decomposes into, why a judge-model can drift, what "the provider is in degraded mode" means) that generalist SREs onboarded cold take months to be useful on-call. The 2024-2026 market signal — covered in chapter 06 — has settled on hiring this specialty directly rather than retraining generalists into it.

Level 35 also sits deliberately *below* the architect / team-lead level (40). You own one platform's reliability contract, not the cross-platform reliability strategy of a team-of-teams; you own the on-call rotation shape, not people management of the rotation members. When those scopes start landing in your queue, the signal is you are ready to level up, not that the charter is wrong.

## Summary

- **One sentence:** AI SRE owns the operational reliability contract of production AI systems.
- **Five surfaces:** SLO engineering, observability, non-deterministic incident response, capacity+vendor risk, chaos+release+DR+program ritual.
- **Four things you do not own:** engine source code, kernel authoring, adversarial defence, the product/prompt/eval surface.
- **Level 35, specialist band:** below architect / team-lead (40), above MLOps (25) and the general infra engineer (20).

The rest of this module makes this concrete: chapter 02 draws the deferral map against each peer; chapter 03 places this role on Dickerson's hierarchy and justifies the module sequence; chapter 04 does the classical-SRE-vs-AI-SRE side-by-side; chapter 05 is the reliability-charter artifact; chapter 06 is the 2024-2026 hiring-market context.
