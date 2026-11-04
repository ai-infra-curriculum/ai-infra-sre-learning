# mod-102 — SRE Foundations Applied to AI

> Second module of the AI Reliability Engineer (AI SRE) track. Rebases the core SRE vocabulary against AI workloads, authors an error-budget policy for non-deterministic systems, designs an on-call rotation for an AI SRE team, builds engine-knob fluency across vLLM / SGLang / TensorRT-LLM / Triton / NVIDIA Dynamo at on-call depth, and makes the engine-authoring peer boundary explicit.

**Estimated effort:** 14 hours
**Position in curriculum:** after `mod-101` (role scope and reliability charter); before `mod-103` (SLO engineering for model-backed services).

## Learning objectives

- Re-ground the core SRE vocabulary against AI workloads — SLI / SLO / error budget, MTTR / MTBF / MTTA / MTTD, toil / toil budget / toil-reduction, embedded vs. consulting SRE models, launch coordination engineering, non-abstract large-system design review.
- Author error-budget policies that work for non-deterministic systems — what a 'bad event' means when the output is a probabilistic string, how to reconcile availability SLO with quality SLO, when to spend budget on canary vs. feature vs. refactor.
- Design an on-call rotation for an AI SRE team — follow-the-sun vs. single-region, primary + secondary + incident-commander roles, hand-off discipline, on-call-health metrics (page volume, actionable rate, sleep disruption), model-aware escalation (kernel SME, eval SME, ML-platform SME, inference-engine SME).
- Develop engine-knob fluency for named inference engines without authoring them — vLLM launch flags, scheduler args, prefix-caching / chunked-prefill knobs, speculative-decoding config; SGLang RadixAttention tuning + data-parallel-attention; TensorRT-LLM engine-build workflow + Triton Inference Server wiring; NVIDIA Dynamo as the disaggregated-serving option — enough to triage a production outage without owning the engine source.
- Cite the boundary to `ai-infra-inference-engineer` (peer, level 35) — this module teaches engine-knob fluency at on-call depth; source-level engine authoring is the peer's primary deliverable.

## Chapters

Read in order. Each chapter is self-contained, 20–30 minutes.

1. [The SRE vocabulary, rebased for AI workloads](01-sre-vocabulary-rebased-for-ai.md) — SLI / SLO / error budget, MTTR / MTBF / MTTA / MTTD, toil, embedded vs. consulting, launch coordination, non-abstract design, all walked for AI examples.
2. [Error-budget policy for non-deterministic systems](02-error-budget-policy-for-non-deterministic-systems.md) — bad-event definitions on probabilistic outputs, reconciliation across availability / latency / quality / cost SLOs, the three-way canary / feature / refactor spend decision.
3. [On-call rotation design for an AI SRE team](03-on-call-rotation-design-for-ai-sre.md) — rotation shape, primary / secondary / IC roles, hand-off discipline, on-call-health metrics, the eight-row model-aware escalation ladder.
4. [vLLM operator fluency — enough to triage, not to author](04-vllm-operator-fluency.md) — PagedAttention, continuous batching, prefix caching; the launch-flag shortlist and the triage flow.
5. [SGLang operator fluency — RadixAttention and the DP-Attention knobs](05-sglang-operator-fluency.md) — radix-tree KV cache, data-parallel attention, the SGLang-vs-vLLM mental model.
6. [TensorRT-LLM, Triton Inference Server, and NVIDIA Dynamo](06-trt-llm-triton-and-dynamo.md) — the engine-build workflow, `config.pbtxt`, the disaggregated-serving option, and the four-engine comparison cheatsheet.
7. [The engine-authoring peer boundary](07-the-engine-authoring-peer-boundary.md) — the two-column contract against `ai-infra-inference-engineer`, the three canonical hand-offs, the ticket template.

## Exercises

Each exercise applies the chapter content to a concrete workload you pick in exercise 01 and carry through. Target: 2 hours per exercise (3 hours for exercise 04).

- [Exercise 01 — SLI / SLO / error-budget vocabulary drill on AI workloads](exercises/exercise-01-sli-slo-error-budget-vocabulary-drill-on-ai-workloads.md) — pick the workload, produce the multi-dimension SLI stack and the stratified MTTR table.
- [Exercise 02 — Error-budget policy authoring for non-deterministic systems](exercises/exercise-02-error-budget-policy-authoring-for-non-deterministic-systems.md) — the seven-section policy document for the workload from exercise 01.
- [Exercise 03 — On-call rotation design for an AI SRE team](exercises/exercise-03-on-call-rotation-design-for-ai-sre.md) — the rotation-design document + hand-off checklist template that enforces the policy from exercise 02.
- [Exercise 04 — vLLM / SGLang / TRT-LLM engine-knob cheatsheet](exercises/exercise-04-vllm-sglang-trt-llm-engine-knob-cheatsheet.md) — one cheatsheet per engine family + a triage runbook routing symptoms to cheatsheets.
- [Exercise 05 — Non-abstract large-system design review of an AI stack](exercises/exercise-05-non-abstract-large-system-design-review-of-an-ai-stack.md) — the eight-section back-of-envelope + residual-risks review artifact.

## Resources

External references cited across the chapters and exercises are collected in [`resources.md`](resources.md).

## Structure

```
mod-102-sre-foundations-applied-to-ai/
├── README.md                                                      ← this file
├── 01-sre-vocabulary-rebased-for-ai.md
├── 02-error-budget-policy-for-non-deterministic-systems.md
├── 03-on-call-rotation-design-for-ai-sre.md
├── 04-vllm-operator-fluency.md
├── 05-sglang-operator-fluency.md
├── 06-trt-llm-triton-and-dynamo.md
├── 07-the-engine-authoring-peer-boundary.md
├── exercises/
│   ├── exercise-01-sli-slo-error-budget-vocabulary-drill-on-ai-workloads.md
│   ├── exercise-02-error-budget-policy-authoring-for-non-deterministic-systems.md
│   ├── exercise-03-on-call-rotation-design-for-ai-sre.md
│   ├── exercise-04-vllm-sglang-trt-llm-engine-knob-cheatsheet.md
│   └── exercise-05-non-abstract-large-system-design-review-of-an-ai-stack.md
├── labs/                                                          ← reserved; no hands-on lab in this module
├── quizzes/                                                       ← reserved; no quiz in this module
└── resources.md
```

## What this module does *not* cover

- **The authored SLO stack for a specific service.** This module teaches SLO vocabulary and policy; `mod-103` authors the full SLI definitions and the per-service numbers.
- **Observability instrumentation and dashboards.** The chapters reference `/metrics` endpoints at operator level; `mod-104` builds the full observability stack.
- **AI-specific failure-class diagnostics in depth.** The on-call ladder names the classes; `mod-105` and `mod-108` provide the per-class runbooks.
- **Capacity planning numbers and fleet sizing math.** This module names the back-of-envelope approach; `mod-106` builds the math.
- **Model-provider risk and fallback contract.** Named in chapter 03 escalation; authored in `mod-107`.
- **Progressive delivery and release gating.** Named in exercise 05; authored in `mod-111`.
- **Engine source-level authoring.** Reserved for the `ai-infra-inference-engineer` peer curriculum.

This module is the *foundation*. The rest of the curriculum deepens it.
