# mod-104-observability-for-llm-and-gpu-workloads: Observability for LLM Request Paths and GPU Workloads

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 20 hours

## Learning objectives

- Instrument an LLM request path with OpenTelemetry using the `gen_ai.*` semantic conventions — `gen_ai.request.model`, `gen_ai.response.id`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reasons`, prompt / completion content capture with PII controls — producing traces that overlay with latency + cost + quality signal
- Instrument the metric layer in Prometheus — TTFT / TPOT / ITL histograms, cache-hit rate, preemption rate, admission-reject rate, queue depth, prefill-decode token mix, worker-concurrency — with PromQL recipes that feed directly into the SLO stack in mod-103
- Instrument GPU telemetry with DCGM-exporter — SM activity, HBM pressure, PCIe / NVLink bandwidth saturation, ECC error counts, XID codes, thermal throttling, power draw — and author the DCGM-aware alerts that distinguish a bad GPU from a bad workload
- Deploy an LLM-trace observability platform — Langfuse / Arize Phoenix / LangSmith as open-source or commercial options — and reason about when to use a dedicated LLM-trace platform vs. OTel-only
- Instrument drift / data-quality signals that read as reliability incidents — Evidently / Arize / WhyLabs — and wire embedding drift, input-distribution shift, and judge-model drift into alerts that page the SRE on-call, not just the ML team
- Author structured log discipline at high QPS — sampling, redaction of prompt / completion for PII, trace-ID correlation, cost-per-log accounting; integrate with Grafana Loki / Cloud Logging
- Cite the boundaries: engine-exposed metrics come from vLLM / SGLang / TRT-LLM (ai-infra-inference-engineer peer owns the exporter surface), kernel-level Nsight / NVTX tracing is at ai-infra-performance peer depth, application-eval trace-overlay is at ai-eval-engineer peer depth

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
