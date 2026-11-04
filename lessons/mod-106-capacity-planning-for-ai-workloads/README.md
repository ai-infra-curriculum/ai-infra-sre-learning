# mod-106-capacity-planning-for-ai-workloads: Capacity Planning for GPU Fleets and Model-Provider Quotas

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 16 hours

## Learning objectives

- Model prefill-bound vs. decode-bound workload profiles with queueing theory — Little's Law applied to request queues, head-of-line blocking, M/M/c as a first-order model, when the simple model breaks (continuous batching, prefix cache, speculative decoding all violate independence) and when to upgrade to simulation
- Forecast GPU-fleet capacity against a mixed-workload traffic shape — chat / RAG / tool-calling / batch mix, prompt-length and completion-length distributions, prefix-cache hit-rate forecast — and translate the forecast into a concrete node-pool shape (count × generation × instance type)
- Plan for GPU-generation diversity in a fleet — Hopper / Ada / Blackwell / GB200 / GB300 / H100 / H200 / MI300 — knowing when a generation swap is a capacity event vs. a performance event, what the cross-generation observability pitfalls are, and how to translate between them
- Plan model-provider quota — OpenAI / Anthropic / Google / Bedrock / Azure tier-and-RPM / TPM budgets, per-key vs. per-org quota pools, quota-reservation vs. burst-credit models, data-residency-constrained provider choice
- Build scale-out / scale-in automation that respects the inference workload — HPA on queue-depth / TTFT-P99 custom metric (not CPU / GPU-util only), Karpenter for GPU bin-packing with spot resilience, warm-pool / scale-to-near-zero for cold-start-sensitive models, Knative as a compare-point
- Run periodic load tests as capacity verification — k6 / Locust-driven traffic shapes mirroring production, Pareto-curve exploration, headroom confirmation; tie load tests into the SLO stack and the chaos program (mod-109)
- Cite the boundaries — GPU-kernel-level throughput calculation at the ai-infra-performance peer; engine-side autoscaling knobs at the ai-infra-inference-engineer peer; base-K8s autoscaling patterns at ai-infra-engineer (level 20)

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
