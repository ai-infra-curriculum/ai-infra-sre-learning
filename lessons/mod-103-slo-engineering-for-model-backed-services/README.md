# mod-103-slo-engineering-for-model-backed-services: SLO Engineering for Model-Backed Services

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 20 hours

## Learning objectives

- Author latency SLOs at per-token granularity — TTFT (time to first token), TPOT (time per output token), ITL (inter-token latency), E2E (end-to-end), goodput — picking percentiles per workload class (chat / batch / RAG / tool-calling / agentic / multimodal) and defending each threshold to the business sponsor
- Author quality SLOs for model-backed services — hallucination rate, toxic-output rate, tool-call-success rate, structured-output-conformance rate, refusal rate, format-drift rate — and compose them with latency SLOs into a single error-budget stack
- Author availability SLOs for streaming responses — 'available' is not a binary when the client sees partial tokens; define partial-success, graceful-degradation, and streaming-stall as distinct SLIs
- Author cost SLOs — cost per 1M input tokens, cost per 1M output tokens, cost per completed request, cost-per-successful-session — tied to the business unit economics and defended against product-ask over-provisioning
- Apply error-budget policies in practice — when to freeze releases on budget burn, when to spend budget on canary, how to weight latency burn vs. quality burn vs. cost burn, how to reconcile with the ai-eval-engineer peer's release gate
- Build an SLO lifecycle — define, baseline against production traffic, instrument, alert, review in the SLO ritual, retire when the service exits scope — and tie each step to a Prometheus / Grafana / OTel artifact
- Cite the boundary to ai-eval-engineer (peer, level 30) — quality SLO *measurement* at the application / eval layer is at that peer; this module owns the SLO as a reliability contract

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
