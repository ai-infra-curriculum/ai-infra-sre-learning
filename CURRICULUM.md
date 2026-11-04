# AI Reliability Engineer (AI SRE) Curriculum

<!-- aicg:orientation:start -->
> **Start here:** [Module 00: Your Relationship to the Model](lessons/00-your-relationship-to-the-model/README.md). Where this role sits relative to the model, what it owns, and who it works with across the sibling curricula.
<!-- aicg:orientation:end -->

**Role level:** 35 (AI-reliability specialist — AI Infrastructure family)
**Planned scope:** 14 modules (216 h) + 3 projects (120 h) = 336 h
**Research evidence:** [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) · [`.aicg/job-requirements.json`](.aicg/job-requirements.json)
**Module + project plan:** [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json)

## Overview

The AI Reliability Engineer / AI SRE owns the **operational reliability contract** of production AI / ML / LLM / agent systems. The role covers:

- **SLO engineering** for model-backed services — quality SLOs (hallucination rate, toxic-output rate, tool-call success rate), latency SLOs at per-token granularity (TTFT / TPOT / ITL), availability under streaming responses, cost per 1 M tokens.
- **Observability** for LLM request paths and GPU workloads — OpenTelemetry `gen_ai.*` traces, DCGM GPU telemetry, Prometheus / Grafana metrics, LLM-trace platforms (Langfuse / Phoenix / LangSmith), drift signals (Arize / WhyLabs / Evidently).
- **Incident response** for non-deterministic failures — silent output drift, hallucination spike, toxic blast, prompt-cache cliff, retrieval drift, agent cascade, judge-model drift, provider-degraded mode.
- **Capacity planning** for GPU fleets and model-provider quotas — queueing-theoretic headroom, cross-generation fleet mix, HPA on queue-depth / TTFT-P99, Karpenter GPU bin-packing.
- **Dependency / vendor risk** on OpenAI / Anthropic / Google / Bedrock / Azure OpenAI — fallback contracts, SLA-vs-SLO reconciliation, status ingestion, vendor eval.
- **Chaos engineering** for AI workloads — GPU eviction, provider timeout injection, cache poisoning, agent-cascade injection, worker-crash under in-flight stream.
- **Release engineering** — canary with response-diffing, shadow traffic, blue-green for large models, SLO-based auto-rollback, DORA four-keys for model releases.
- **Data and config pipelines** as reliability surface — prompt-template versioning, retrieval-freshness SLOs, feature drift, eval-set drift.
- **DR / BCP / multi-region** for AI workloads — NIST SP 800-34 applied, stateful-agent state reconstitution, data-residency-respecting failover.
- **Cross-functional reliability leadership** at team scope — SLO review ritual, on-call shape, build-vs-buy of SRE tooling, cross-peer interface contracts.

## Ownership rule

Assign primary coverage to the lowest-level role that genuinely requires the skill. Higher-level tracks link to the owner rather than duplicate content.

This track owns the **reliability contract** of production AI systems. It is a **peer at the same level** to three other AI-Infrastructure specialists (all level 35) and divides ownership by primary deliverable:

| Track | Owns |
|---|---|
| `ai-infra-inference-engineer` (peer, level 35) | the serving system itself — vLLM / SGLang / TRT-LLM / Dynamo at source-level depth, request scheduling, KV-cache runtime management |
| `ai-infra-performance` (peer, level 35) | the kernel-authoring / GPU-optimisation craft — CUDA / CUTLASS / CuTe DSL / Nsight-Compute-to-SASS |
| `ai-infra-security` (peer, level 35) | inference-edge adversarial defence — prompt-injection, PII sidecars, model-extraction defence, ML-BOM signing |
| **`ai-infra-sre` (this track)** | the reliability contract around the stack — SLO, incident, chaos, capacity, vendor risk |

This track defers *down* to `ai-infra-ml-platform` (peer, level 30), `ai-eval-engineer` (peer, level 30), `model-evaluation-engineer` (peer, level 30), `ai-infra-mlops` (peer, level 25), the AI Engineering consumers (`llm-application-developer`, `rag-engineer`, `applied-ai-engineer` at level 25), `ai-infra-engineer` (level 20), `ai-infra-junior-engineer` (level 10). It defers *up* to `ai-infra-architect` / `ai-infra-team-lead` (level 40), `ai-infra-senior-architect` (level 45), and `ai-infra-principal-engineer` / `ai-infra-principal-architect` (level 50). Full ownership-rule text is in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json) and [`.aicg/job-requirements.json`](.aicg/job-requirements.json).

## Module plan

| ID | Title | Hours | Status |
|---|---|---|---|
| [`mod-101-ai-sre-position-on-the-ladder`](lessons/mod-101-ai-sre-position-on-the-ladder) | The AI SRE Position on the Infra Ladder | 10 | planned |
| [`mod-102-sre-foundations-applied-to-ai`](lessons/mod-102-sre-foundations-applied-to-ai) | SRE Foundations Applied to AI — Vocabulary, Methodology, and Engine Literacy | 14 | planned |
| [`mod-103-slo-engineering-for-model-backed-services`](lessons/mod-103-slo-engineering-for-model-backed-services) | SLO Engineering for Model-Backed Services | 20 | planned |
| [`mod-104-observability-for-llm-and-gpu-workloads`](lessons/mod-104-observability-for-llm-and-gpu-workloads) | Observability for LLM Request Paths and GPU Workloads | 20 | planned |
| [`mod-105-non-deterministic-failure-diagnosis`](lessons/mod-105-non-deterministic-failure-diagnosis) | Non-Deterministic Failures — Diagnosing AI-Specific Incidents | 18 | planned |
| [`mod-106-capacity-planning-for-ai-workloads`](lessons/mod-106-capacity-planning-for-ai-workloads) | Capacity Planning for GPU Fleets and Model-Provider Quotas | 16 | planned |
| [`mod-107-vendor-and-model-provider-risk`](lessons/mod-107-vendor-and-model-provider-risk) | Vendor and Model-Provider Dependency Risk | 14 | planned |
| [`mod-108-incident-response-for-ai-systems`](lessons/mod-108-incident-response-for-ai-systems) | Incident Response, Runbooks, and Post-Mortem Culture for AI Systems | 18 | planned |
| [`mod-109-chaos-engineering-for-ai`](lessons/mod-109-chaos-engineering-for-ai) | Chaos Engineering for AI Workloads | 16 | planned |
| [`mod-110-security-safety-and-compliance-reliability-crossover`](lessons/mod-110-security-safety-and-compliance-reliability-crossover) | Reliability Crossover with Security, Safety, and Compliance | 12 | planned |
| [`mod-111-release-engineering-and-progressive-delivery`](lessons/mod-111-release-engineering-and-progressive-delivery) | Release Engineering and Progressive Delivery for Model Updates | 16 | planned |
| [`mod-112-data-and-config-pipelines-as-reliability-surface`](lessons/mod-112-data-and-config-pipelines-as-reliability-surface) | Data and Config Pipelines as Reliability Surface | 14 | planned |
| [`mod-113-dr-bcp-and-multi-region-ai`](lessons/mod-113-dr-bcp-and-multi-region-ai) | DR, BCP, and Multi-Region for AI Workloads | 14 | planned |
| [`mod-114-cross-functional-reliability-leadership`](lessons/mod-114-cross-functional-reliability-leadership) | Cross-Functional Reliability Leadership at Team Scope | 14 | planned |

**Module hours total:** 216

## Project plan

| ID | Title | Hours | Status |
|---|---|---|---|
| [`project-101-slo-and-observability-stack-for-a-single-fleet`](projects/project-101-slo-and-observability-stack-for-a-single-fleet) | SLO + Observability Stack for a Single Inference Fleet | 40 | planned |
| [`project-102-ai-incident-and-chaos-program`](projects/project-102-ai-incident-and-chaos-program) | AI Incident + Chaos Program for an Inference Platform | 40 | planned |
| [`project-103-release-and-dr-program`](projects/project-103-release-and-dr-program) | Release + DR / BCP Program for Multi-Region AI | 40 | planned |

**Project hours total:** 120
**Grand total:** 336 h

## Per-module summary

### mod-101 — The AI SRE Position on the Infra Ladder (10 h)

Positions AI SRE against its three level-35 peers (`ai-infra-inference-engineer`, `ai-infra-performance`, `ai-infra-security`), the level-30 peers (`ai-infra-ml-platform`, `ai-eval-engineer`, `model-evaluation-engineer`), and the consumer tracks. Grounds the role in Dickerson's hierarchy and produces the reliability charter that routes any incoming concern to the right owner.

### mod-102 — SRE Foundations Applied to AI (14 h)

Re-grounds core SRE vocabulary (SLI / SLO / error budget, MTTR / MTBF, toil, on-call design, launch coordination, non-abstract design review) against AI-specific workloads. Builds engine-knob fluency for vLLM / SGLang / TRT-LLM / Dynamo at **on-call depth** — enough to triage without authoring the engine.

### mod-103 — SLO Engineering for Model-Backed Services (20 h)

The core-of-core module. Latency SLOs at per-token granularity (TTFT / TPOT / ITL / E2E / goodput). Quality SLOs (hallucination, toxic output, tool-call success, structured-output conformance, refusal). Availability under streaming. Cost per 1 M tokens. Error-budget policy for non-deterministic systems. SLO lifecycle end-to-end.

### mod-104 — Observability for LLM Request Paths and GPU Workloads (20 h)

OpenTelemetry `gen_ai.*` semantic conventions. Prometheus PromQL recipes for TTFT / TPOT / ITL / cache-hit / preemption. DCGM-exporter for GPU telemetry (SM activity, HBM pressure, ECC, XID, thermal). LLM-trace platforms (Langfuse / Phoenix / LangSmith). Drift signals (Arize / WhyLabs / Evidently) wired as SRE alerts. Structured log discipline at high QPS with PII redaction.

### mod-105 — Non-Deterministic Failure Diagnosis (18 h)

The failure-class catalog that separates AI SRE from classical SRE. Silent output drift, hallucination spike, toxic blast, prompt-cache cliff, retrieval drift, agent cascade, judge-model drift, provider-degraded mode. Detection discipline for each class. Triage decision matrix that routes incidents to the right owner.

### mod-106 — Capacity Planning for GPU Fleets and Model-Provider Quotas (16 h)

Queueing-theoretic headroom for prefill-bound vs. decode-bound workloads. Cross-generation fleet planning (H100 / H200 / B200 / GB200 / GB300 / MI300). Model-provider quota planning. HPA on inference-specific custom metrics. Karpenter GPU bin-packing. Load testing with k6 / Locust as capacity verification.

### mod-107 — Vendor and Model-Provider Dependency Risk (14 h)

Vendor-dependency surface mapping. Provider-fallback contracts (LiteLLM / Portkey shape). Status ingestion into the incident-response pipeline. SLA-vs-SLO reconciliation. Public-incident case studies (OpenAI Nov 2023 DDoS, subsequent frontier-lab outages). Vendor eval discipline.

### mod-108 — Incident Response for AI Systems (18 h)

The top-twenty AI-specific runbooks. Incident command adapted for AI systems. Blameless post-mortems with AI-flavored root-cause categories. Regression-fixture back-feed discipline. Legal / compliance / regulator handoff for AI-safety-adjacent incidents.

### mod-109 — Chaos Engineering for AI Workloads (16 h)

Principles of Chaos Engineering extended to AI. Chaos Mesh / LitmusChaos / Gremlin / AWS FIS as tooling. AI-specific experiments — GPU eviction, provider timeout, cache poisoning, agent-cascade injection. Chaos-test the provider-fallback contract. Chaos as a continuous practice.

### mod-110 — Reliability Crossover with Security, Safety, and Compliance (12 h)

The reliability dimension of defensive controls — guardrail-sidecar latency budget, PII-filter cache impact, model-extraction query-rate control impact on P99. Joint post-mortems with the `ai-infra-security` peer. Compliance telemetry hooks (EU AI Act Article 72, FDA PCCP, SR 11-7). Reliability-gate vs. safety-gate reconciliation.

### mod-111 — Release Engineering and Progressive Delivery (16 h)

Canary with response-diffing + semantic-equivalence tolerance. Shadow traffic for model changes. Blue-green for large models. SLO-based auto-rollback. Joint eval + SLO release gate. DORA four-keys for model releases.

### mod-112 — Data and Config Pipelines as Reliability Surface (14 h)

Prompt-template versioning + rollback. Retrieval-freshness SLOs for RAG. Feature drift as serving incident. Eval-set drift and rot. Unified pipeline observability in a single pane.

### mod-113 — DR, BCP, and Multi-Region for AI Workloads (14 h)

NIST SP 800-34 applied to AI. Regional-outage playbooks. Stateful-agent state reconstitution. Data-residency-respecting failover. DR rehearsal schedule (table-top to full failover). BCP artifact pack for regulated-industry audits.

### mod-114 — Cross-Functional Reliability Leadership at Team Scope (14 h)

Reliability program cadence + SLO-review ritual. Build-vs-buy RFC for the SRE-tooling stack. Cross-peer interface contracts. Reliability communication upward. AI-SRE career trajectory + certification portfolio (CKA / CKAD, NVIDIA DLI, CISSP for BCP / IR credibility, cloud-provider reliability specialty).

## Status

> **Scaffolded by `aicg org bootstrap-role`.** The research + plan artifacts in `.aicg/` are populated; module and project content are not yet authored. Run `aicg org daily` to drive the autonomous fill-in loop.
>
> **Research cycle (2026-11-04):** scaffold-mode (live postings deferred — WebSearch / WebFetch permission-gated). See [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) Status section.
