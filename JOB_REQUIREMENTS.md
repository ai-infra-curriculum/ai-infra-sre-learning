# Job Requirements — AI Reliability Engineer (AI SRE)

**Role level:** 35 (AI-reliability specialist — AI Infrastructure family)
**Track:** `ai-infra-sre-learning`
**Research window:** 2026-08-06 → 2026-11-04 (last 90 days)
**Today:** 2026-11-04
**Live postings sampled this cycle:** 0 — WebSearch / WebFetch permission-gated; see Status below

This file documents the requirements catalog used to seed the AI Reliability Engineer curriculum. Raw normalised data lives in [`.aicg/job-requirements.json`](.aicg/job-requirements.json); the planned curriculum lives in [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json).

## Status — 2026-11 bootstrap scaffold, live postings deferred

The 2026-11-04 bootstrap session attempted WebSearch and WebFetch to fan requirement discovery across Greenhouse / Lever / Ashby / Workday / employer career sites — both returned the explicit `Claude requested permissions ... but you haven't granted it yet.` error. Delegated subagents inherit the same permission state and cannot exercise either tool either. The `postings` array in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) is therefore left empty and every `requirement_themes[].frequency` is marked `"needs-research"`.

Rather than invent posting evidence, this scaffold grounds every requirement theme in **authoritative public references** — the Google SRE books, NIST AI RMF Generative AI Profile, OpenTelemetry GenAI semantic conventions, DCGM / Prometheus / Grafana docs, Chaos Engineering standards, public frontier-lab incident write-ups, and the sibling `ai-infra-performance-learning` packet (63 in-window postings for the peer kernel-authoring track as of 2026-10). The autonomous research loop should backfill `postings[]` on first run and recompute `requirement_themes[].frequency` against the realised n.

**Discrepancy flagged for reconciliation:** the sibling `ai-infra-inference-engineer-learning` packet (authored before this one) lists `ai-infra-sre` at **level 30**. This packet is authored at **level 35** per the explicit directive of the bootstrap task, and level 35 is the right placement given the emerging 2026 AI-SRE market signal — Anthropic Reliability, OpenAI Applied, Google Vertex SRE, Databricks Mosaic Reliability, Fireworks / Together / Baseten Production Engineering hire at a mid-to-staff engineer band with AI-specific scope, not a generalist SRE band. The autonomous loop should reconcile sibling `peer_and_higher_level_roles[]` blocks on its next pass.

Authoritative public references grounding every requirement theme:

- **SRE canon** — [Google SRE Book](https://sre.google/sre-book/table-of-contents/), [Google SRE Workbook](https://sre.google/workbook/table-of-contents/), [Building Secure and Reliable Systems](https://sre.google/books/building-secure-reliable-systems/), [Dickerson's Hierarchy of Reliability](https://sre.google/sre-book/part-III-practices/), [The Tail at Scale (Dean & Barroso)](https://research.google/pubs/pub40801/), [Performance Modelling and Design of Computer Systems (Harchol-Balter)](https://www.cs.cmu.edu/~harchol/PerformanceModeling/book.html), [Blameless Post-Mortem Culture](https://sre.google/sre-book/postmortem-culture/).
- **Observability + GPU telemetry** — [OpenTelemetry specification + semantic conventions](https://opentelemetry.io/docs/), [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/), [Prometheus / PromQL](https://prometheus.io/docs/), [Grafana](https://grafana.com/docs/), [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/), [NVIDIA XID error reference](https://docs.nvidia.com/deploy/xid-errors/index.html), [NVIDIA GPU Operator on Kubernetes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/), [Kubernetes Node Problem Detector](https://github.com/kubernetes/node-problem-detector), [Arize AI](https://docs.arize.com/), [WhyLabs](https://docs.whylabs.ai/), [Evidently AI](https://docs.evidentlyai.com/), [LangSmith](https://docs.smith.langchain.com/), [Langfuse](https://langfuse.com/docs), [Arize Phoenix](https://arize.com/docs/phoenix).
- **Non-deterministic failure taxonomy** — [NIST AI RMF Generative AI Profile (AI 600-1)](https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile), [AI Incident Database](https://incidentdatabase.ai/), [OECD.AI AI Incidents Monitor](https://oecd.ai/en/incidents), [MIT AI Risk Repository](https://airisk.mit.edu/).
- **Chaos engineering** — [Principles of Chaos Engineering](https://principlesofchaos.org/), [Chaos Mesh](https://chaos-mesh.org/docs/), [LitmusChaos](https://litmuschaos.io/docs/), [Gremlin](https://www.gremlin.com/docs/), [AWS Fault Injection Service](https://docs.aws.amazon.com/fis/).
- **Progressive delivery + release engineering** — [Flagger](https://docs.flagger.app/), [Argo Rollouts](https://argoproj.github.io/argo-rollouts/), [Istio](https://istio.io/latest/docs/), [DORA Four Keys](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance), [Google Cloud MLOps continuous delivery](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning).
- **Kubernetes reliability for AI workloads** — [Kubernetes probes, PDBs, graceful shutdown](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/), [Karpenter](https://karpenter.sh/docs/), [Knative Serving](https://knative.dev/docs/serving/).
- **Capacity + load testing** — [k6](https://k6.io/docs/), [Locust](https://docs.locust.io/), [Kubecost / OpenCost](https://docs.kubecost.com/).
- **Vendor / provider dependency** — [OpenAI Status](https://status.openai.com/) + the November 2023 DDoS incident write-up, [Anthropic Status](https://status.anthropic.com/), [Google Cloud / Vertex Status](https://status.cloud.google.com/), [AWS Service Health + Bedrock status](https://health.aws.amazon.com/), [Azure Status](https://status.azure.com/), [LiteLLM](https://docs.litellm.ai/), [Portkey](https://docs.portkey.ai/), [OpenAI Cookbook — production rate-limit patterns](https://cookbook.openai.com/examples/how_to_handle_rate_limits).
- **DR / BCP** — [NIST SP 800-34 Rev 1](https://csrc.nist.gov/publications/detail/sp/800-34/rev-1/final), [AWS Well-Architected Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html), [Azure Reliability Principles](https://learn.microsoft.com/en-us/azure/well-architected/reliability/principles), [Google Cloud Reliability Guide](https://cloud.google.com/architecture/framework/reliability), [(ISC)² CISSP — Business Continuity + Incident Response domains](https://www.isc2.org/Certifications/CISSP).
- **Inference-engine landscape (operator-depth)** — [vLLM](https://docs.vllm.ai/), [SGLang](https://docs.sglang.ai/), [NVIDIA TensorRT-LLM](https://nvidia.github.io/TensorRT-LLM/), [NVIDIA Triton Inference Server](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/), [NVIDIA Dynamo](https://docs.nvidia.com/dynamo/).
- **Safety / responsible-AI release tiering** — [Anthropic Responsible Scaling Policy](https://www.anthropic.com/rsp), [OpenAI Preparedness Framework](https://openai.com/safety/preparedness/), [Google DeepMind Frontier Safety Framework](https://deepmind.google/discover/blog/introducing-the-frontier-safety-framework/).
- **Certifications and training** — [CNCF CKA / CKAD](https://www.cncf.io/training/certification/), [NVIDIA Deep Learning Institute](https://www.nvidia.com/en-us/training/), [(ISC)² CISSP](https://www.isc2.org/Certifications/CISSP), cloud-provider reliability specialty certifications.

Every requirement theme below cites at least one of these references, and every theme is shaped so posting-frequency evidence can be added underneath it without restructure.

## Methodology

1. Scoped the canonical task domains for an AI SRE (level 35, AI Infrastructure family) from the authoritative references above and the scope cues in the bootstrap task (SLOs for model-backed services, observability for LLM + GPU workloads, incident response for non-deterministic failures, capacity + vendor risk, chaos testing).
2. Mapped each task domain to (a) the role on the level ladder that should own it primarily and (b) the curriculum module that will cover it.
3. Applied the **ownership rule** — assign coverage to the lowest-level role that genuinely requires the skill; higher-level tracks link back rather than duplicate. In particular, deferred:
   - *Sideways at the same level* to `ai-infra-inference-engineer` (peer, level 35) for inference-engine source-level internals and authoring depth — this track teaches engine-knob fluency only.
   - *Sideways at the same level* to `ai-infra-performance` (peer, level 35) for CUDA / kernel-authoring / Nsight-to-SASS / CUTLASS depth.
   - *Sideways at the same level* to `ai-infra-security` (peer, level 35) for inference-edge adversarial defence depth.
   - *Down* to `ai-infra-ml-platform` (peer, level 30) for the self-service ML platform / model-registry handoff.
   - *Down* to `ai-eval-engineer` (peer, level 30, AI Engineering family) for application-layer eval release-gate depth.
   - *Down* to `model-evaluation-engineer` (peer, level 30, ML Engineering family) for benchmark / statistical / calibration methodology.
   - *Down* to `ai-infra-mlops` (peer, level 25) for CI/CD automation this role plugs reliability gates into.
   - *Down* to `ai-infra-engineer` (level 20) for K8s / Terraform / cloud / Docker / observability base.
   - *Down* to `ai-infra-junior-engineer` (level 10) for engineering-craft prerequisites.
   - *Up* to `ai-infra-architect` (level 40) for cross-platform reliability architecture at team-of-teams scope and `ai-infra-team-lead` (level 40) for people-management scope.
   - *Up* to `ai-infra-senior-architect` (level 45) for organisation-scope reliability strategy.
   - *Up* to `ai-infra-principal-engineer` / `ai-infra-principal-architect` (level 50) for principal scope.
   - *Out of scope* to the AI Engineering family (prompt engineering, agent design, product analytics), to legal / compliance / privacy for licensing / data-residency / regulatory sign-off, to classical SRE for non-AI systems the organisation runs, and to SecOps / DFIR / IR for security-incident evidence handling (this role contributes AI-specific telemetry but is not the system of record).
4. Flagged every requirement frequency as `<!-- needs-research -->` so the next cycle can either confirm, demote, or promote based on live evidence.

## Requirement themes → curriculum ownership

The table below lists each requirement theme, its posting frequency this cycle (not yet sampled — see Status), its planned owner per the level hierarchy, and the curriculum coverage path.

| # | Theme | Freq | Owner role | Coverage |
|---|---|---|---|---|
| 1 | SLO / SLI / error-budget engineering specific to model-backed services (quality SLO, per-token latency SLO, streaming-availability, cost SLO) | <!-- needs-research --> | `ai-infra-sre` (this) | [`mod-103-slo-engineering-for-model-backed-services`](lessons/mod-103-slo-engineering-for-model-backed-services) |
| 2 | Observability for LLM request paths + GPU workloads (OTel `gen_ai.*`, Prometheus / Grafana, DCGM, Langfuse / Phoenix / LangSmith, drift via Arize / WhyLabs / Evidently) | <!-- needs-research --> | `ai-infra-sre` | [`mod-104-observability-for-llm-and-gpu-workloads`](lessons/mod-104-observability-for-llm-and-gpu-workloads) |
| 3 | Incident response for non-deterministic AI failures (silent drift, hallucination spike, toxic blast, prompt-cache cliff, retrieval drift, agent cascade, judge-model drift, provider degraded) | <!-- needs-research --> | `ai-infra-sre` | [`mod-105-non-deterministic-failure-diagnosis`](lessons/mod-105-non-deterministic-failure-diagnosis) |
| 4 | Incident response, runbooks, and post-mortem culture adapted to AI systems | <!-- needs-research --> | `ai-infra-sre` | [`mod-108-incident-response-for-ai-systems`](lessons/mod-108-incident-response-for-ai-systems) |
| 5 | Capacity planning for GPU fleets + model-provider quotas (prefill-vs-decode queueing, cross-generation fleet mix, HPA on queue-depth / TTFT-P99, Karpenter GPU bin-packing) | <!-- needs-research --> | `ai-infra-sre` | [`mod-106-capacity-planning-for-ai-workloads`](lessons/mod-106-capacity-planning-for-ai-workloads) |
| 6 | Dependency / vendor risk on OpenAI / Anthropic / Google / Bedrock / Azure OpenAI (fallback contract, SLA-vs-SLO reconciliation, status ingestion, vendor eval) | <!-- needs-research --> | `ai-infra-sre` | [`mod-107-vendor-and-model-provider-risk`](lessons/mod-107-vendor-and-model-provider-risk) |
| 7 | Chaos engineering targeted at AI workloads (GPU eviction, provider timeout, cache poisoning, agent cascade, worker crash mid-stream) | <!-- needs-research --> | `ai-infra-sre` | [`mod-109-chaos-engineering-for-ai`](lessons/mod-109-chaos-engineering-for-ai) |
| 8 | Release engineering + progressive delivery for model updates (Flagger / Argo Rollouts, canary + shadow + blue-green with SLO-based auto-rollback, DORA four-keys for model releases) | <!-- needs-research --> | `ai-infra-sre` | [`mod-111-release-engineering-and-progressive-delivery`](lessons/mod-111-release-engineering-and-progressive-delivery) |
| 9 | Data + config pipelines as reliability surface (prompt-template versioning, retrieval-freshness SLOs, feature drift, eval-set drift) | <!-- needs-research --> | `ai-infra-sre` | [`mod-112-data-and-config-pipelines-as-reliability-surface`](lessons/mod-112-data-and-config-pipelines-as-reliability-surface) |
| 10 | DR / BCP / multi-region for AI workloads (NIST SP 800-34 applied, stateful-agent reconstitution, data-residency-respecting failover, DR rehearsal schedule) | <!-- needs-research --> | `ai-infra-sre` | [`mod-113-dr-bcp-and-multi-region-ai`](lessons/mod-113-dr-bcp-and-multi-region-ai) |
| 11 | Reliability crossover with security / safety / compliance (guardrail latency budget, joint post-mortems, EU AI Act Article 72 / FDA PCCP / SR 11-7 telemetry hooks) | <!-- needs-research --> | `ai-infra-sre` | [`mod-110-security-safety-and-compliance-reliability-crossover`](lessons/mod-110-security-safety-and-compliance-reliability-crossover) |
| 12 | Cross-functional reliability program leadership at team scope (SLO ritual, on-call shape, build-vs-buy of SRE tooling, cross-peer interface contracts) | <!-- needs-research --> | `ai-infra-sre` | [`mod-114-cross-functional-reliability-leadership`](lessons/mod-114-cross-functional-reliability-leadership) |
| 13 | Classical SRE foundations (error budgets, toil, on-call, launch coordination, non-abstract design review) applied to AI systems | <!-- needs-research --> | `ai-infra-sre` | [`mod-102-sre-foundations-applied-to-ai`](lessons/mod-102-sre-foundations-applied-to-ai) |
| 14 | Role scope + deferral contract vs. peer and higher-level tracks | <!-- needs-research --> | `ai-infra-sre` | [`mod-101-ai-sre-position-on-the-ladder`](lessons/mod-101-ai-sre-position-on-the-ladder) |
| 15 | Operational literacy with named inference engines (vLLM, SGLang, TRT-LLM, Dynamo, Triton) at engine-knob-fluency depth | <!-- needs-research --> | `ai-infra-sre` | [`mod-102-sre-foundations-applied-to-ai`](lessons/mod-102-sre-foundations-applied-to-ai) + threaded through mod-103, 104, 105, 108 |
| 16 | GPU / CUDA / kernel-authoring / CUTLASS / Nsight-to-SASS depth | <!-- needs-research --> | `ai-infra-performance` (peer, level 35) | Linked out — this track reads GPU telemetry at triage depth only |
| 17 | Inference-engine source-level internals + authoring | <!-- needs-research --> | `ai-infra-inference-engineer` (peer, level 35) | Linked out — this track operates engines but does not author them |
| 18 | Inference-edge adversarial defence depth (prompt-injection, PII sidecar, model-extraction defence, ML-BOM signing) | <!-- needs-research --> | `ai-infra-security` (peer, level 35) | Linked out — this track owns the reliability knock-on of defensive controls only |
| 19 | MLOps CI/CD pipeline authoring | <!-- needs-research --> | `ai-infra-mlops` (peer, level 25) | Linked out — this track plugs reliability gates into their pipelines |
| 20 | Model-artifact handoff from the ML platform (safetensors / GGUF / TRT-engine, lineage, eval evidence) | <!-- needs-research --> | `ai-infra-ml-platform` (peer, level 30) | Linked out — this track consumes the handoff contract |
| 21 | Application / model-eval release-gate methodology | <!-- needs-research --> | `ai-eval-engineer` (peer, level 30) | Linked out — this track consumes the gate signal and wires it into auto-rollback |
| 22 | K8s / Terraform / cloud / Docker / observability base | <!-- needs-research --> | `ai-infra-engineer` (level 20) | Prerequisite — listed in [`PREREQUISITES.md`](PREREQUISITES.md); GPU-specific reliability extensions are covered here |
| 23 | Engineering-craft prerequisites (Linux, Git, Python, Bash, HTTP, networking primitives) | <!-- needs-research --> | `ai-infra-junior-engineer` (level 10) | Prerequisite — listed in [`PREREQUISITES.md`](PREREQUISITES.md) |
| 24 | Cross-platform reliability architecture + organisation-scope strategy | <!-- needs-research --> | `ai-infra-architect` (level 40) | Linked out — this track feeds RFCs upward |
| 25 | People management for an AI SRE / production-engineering team | <!-- needs-research --> | `ai-infra-team-lead` (level 40) | Linked out — this track stops at technical-leadership scope |

## Salary signal

Not sampled this cycle — see `salary_summary` in [`.aicg/job-requirements.json`](.aicg/job-requirements.json) for the sibling-cycle hint and the sampling convention. <!-- needs-research: live salary distribution across ≥15 postings with declared ranges -->

## What to expect on the next research cycle

- Fan WebSearch + WebFetch across the targets catalogued in `research_status.needs_research_note` of [`.aicg/job-requirements.json`](.aicg/job-requirements.json) — hyperscaler and frontier-lab reliability teams, inference-provider Production Engineering, enterprise AI-SRE in regulated industries, AI-first SaaS reliability teams, and the ATS aggregators (Greenhouse / Lever / Ashby / Workday / SmartRecruiters) under the title-fragment set.
- Backfill `postings[]`, recompute `requirement_themes[].frequency` against the realised n, and update the frequency column in the table above.
- Reconcile the peer-level discrepancy with the sibling `ai-infra-inference-engineer-learning` packet (which lists `ai-infra-sre` at level 30). Level 35 is this packet's authoritative placement per the bootstrap-task directive; the next loop should update the sibling peer blocks.
- Flag themes whose evidence stays empty for demotion; promote any emerging theme (agentic-workload reliability, MCP-server-side reliability, multimodal / voice / video reliability) that clears a 0.30 frequency threshold.
