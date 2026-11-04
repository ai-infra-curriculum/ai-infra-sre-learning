# The SRE Vocabulary, Rebased for AI Workloads

## Why this chapter exists

You will spend the rest of this track reading documents that assume you can translate the classical SRE vocabulary — SLI, SLO, error budget, MTTR / MTBF / MTTA / MTTD, toil, embedded vs. consulting, launch coordination, non-abstract design review — into the AI surface without pause. This chapter is the translation key. Nothing here is new terminology; what is new is the AI-shaped example sitting underneath each term.

The reference for the classical definitions is the [Google SRE Book](https://sre.google/sre-book/table-of-contents/) and the [SRE Workbook](https://sre.google/workbook/table-of-contents/). The AI-side examples are sourced from the remaining modules in this curriculum and will be deepened in `mod-103` (SLOs), `mod-104` (observability), `mod-108` (incident response), and `mod-114` (program ritual).

## 1. SLI / SLO / error budget — rebased

### Classical definitions

- **SLI (Service Level Indicator).** A directly measured ratio of good events to valid events over a window — "fraction of requests completing in ≤ 300 ms," "fraction of requests returning 2xx or 3xx." Defined in [SRE Workbook, Implementing SLOs](https://sre.google/workbook/implementing-slos/).
- **SLO (Service Level Objective).** A target for the SLI over a stated window — "99.9% of eligible requests complete in ≤ 300 ms, measured over 28 rolling days."
- **Error budget.** The complement of the SLO — "0.1% of eligible requests may complete in > 300 ms over 28 rolling days." Policy says what happens when the budget is burned.

### What rebases for AI workloads

- **Event granularity changes.** A single logical "request" to a streaming LLM endpoint is a stream of tokens. You can author per-request SLIs (classical), or per-token SLIs on TTFT / TPOT / ITL (AI-specific). The decision is which contract the user is actually experiencing. Covered depthwise in `mod-103`.
- **"Good event" definition becomes multi-dimensional.** A classical event is good if it is fast and did not 5xx. An AI event is good if it is fast *and* did not 5xx *and* did not hallucinate *and* did not over-refuse *and* produced valid structured output. The error-budget stack has to resolve across those dimensions (chapter 02 covers the arithmetic).
- **"Valid event" definition is load-bearing.** Classical SRE care which requests count as valid (health-check noise, abandoned connections, admission-controlled). For AI, you additionally have to decide whether safety-refusals count as valid events, whether fallback-provider responses count against the primary SLO, whether cached prefill hits are excluded from TTFT measurement. Every one of those decisions is a review item in the SLO document.

### A minimal AI SLI stack (teaser — authored in `mod-103`)

| Dimension | Example SLI | Notes |
|---|---|---|
| Availability | 2xx-or-3xx rate on `/v1/chat/completions` | Classical shape |
| Latency — TTFT | p99 ≤ 500 ms for streaming requests | Decomposes classical request-latency |
| Latency — TPOT | p99 ≤ 50 ms/token mid-stream | Steady-state generation |
| Latency — ITL stability | p99 inter-token gap ≤ 100 ms | Stall detection |
| Quality | judge-model pass-rate on golden-stimulus set ≥ 95% | Signal constructed by eval peer |
| Cost | cost per 1M successful tokens ≤ budget | Promoted to SLI, not budget |

## 2. MTTR / MTBF / MTTA / MTTD — rebased

### Classical definitions

- **MTTD — Mean Time To Detect.** From fault occurrence to the monitoring system firing.
- **MTTA — Mean Time To Acknowledge.** From alert to on-call pickup.
- **MTTR — Mean Time To Recover / Remediate / Repair** (the industry is sloppy here — document which you mean in your postmortem template). From fault occurrence to service restored to SLO.
- **MTBF — Mean Time Between Failures.** Reliability of the system in the aggregate.

### What rebases for AI workloads

**MTTD decomposes by failure class.** Classical MTTD is a single number — the time until any signal fires. For AI, the MTTD depends on whether the fault is a classical infrastructure fault (seconds), a quality regression (minutes to hours, bounded by the judge-pass-rate signal window), or a drift-style silent failure (sometimes days — covered in `mod-105`). Report MTTD as a stratified number. A platform with a one-minute MTTD for 5xx and a four-day MTTD for a hallucination spike is a different platform from one with a one-minute MTTD across both.

**MTTR has a prompt / config axis.** Classical MTTR is "revert the binary, flush the cache, restore from backup." For AI, add: revert the prompt template, re-pin a retrieval index snapshot, fall back to a prior model artifact, flip a provider in the LiteLLM / Portkey router, disable a guardrail sidecar. Each is a revert path and each needs its own time-to-execute measured (`mod-108`).

**MTBF is a weak signal on AI surfaces.** The right-tail of AI failures is dominated by rare, high-impact events (a provider's model being deprecated and silently falling back, a drift that only compounds past a week). A monthly MTBF number hides them. Prefer stratified failure-class MTBF ("hallucination spike: 1 per quarter, latency-SLO breach: 2 per month, GPU XID 79: 1 per fortnight") over a single aggregate.

**MTTA is unchanged — but the paging policy around it is not.** Classical paging pages on burn rate, not single-sample excursion. AI quality signals are noisy by construction; use multi-window / multi-burn-rate alerting (see [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)) on the constructed quality signals the same way you would on request-error-rate. Covered in `mod-104`.

## 3. Toil, toil budget, toil reduction — rebased

### Classical definition

[SRE Book, Chapter 5 — Eliminating Toil](https://sre.google/sre-book/eliminating-toil/): toil is **manual, repetitive, automatable, tactical (not strategic), interrupt-driven, scaling linearly with service growth, and devoid of enduring value**. Google's SRE org caps toil at ≤ 50% of an SRE engineer's time.

### What rebases for AI workloads

Toil that *looks* like engineering work but is actually toil on AI surfaces:

- **Manual canary analysis** — eyeballing a judge-model pass-rate dashboard on a rollout because the auto-promote gate was never wired. Classical analogue: manual error-rate review on a deploy. Automation: wire the canary to the semantic-diff / judge signal (`mod-111`).
- **Manual quota ticket filing** — opening a weekly ticket with OpenAI / Anthropic / Bedrock to raise an RPM / TPM cap because usage is growing. Classical analogue: manual IAM ticket for a cloud quota. Automation: forecast the quota headroom off a rolling demand signal and open the ticket automatically with a growth projection, or auto-fail over to the secondary provider under predicted pressure (`mod-106` + `mod-107`).
- **Manual drift chart review** — a ten-minute daily ritual of scanning Arize / Phoenix dashboards for anything odd. Classical analogue: manual log-tail review. Automation: wire the drift statistic to burn-rate alerting against a quality SLO so the dashboard only matters when the alert fires (`mod-104` + `mod-105`).
- **Manual model promotion** — copy-paste from a training run to a serving-registry name, hand-edit the model card, kick the deploy. Classical analogue: manual binary rollout. Automation: a model-promotion pipeline with eval-gate (peer-owned by `ai-infra-ml-platform`, consumed by you).
- **Manual prompt hotfix** — editing a prompt template in a config repo without versioned rollback. Classical analogue: edit-the-config-file deploy. Automation: prompt-and-config registry treated as a release surface (`mod-112`).
- **Manual GPU reboot** — SSH to a node, run `nvidia-smi -r`, mark the node drained, rotate it back. Classical analogue: manual host reboot. Automation: Node Problem Detector + the NVIDIA GPU Operator's health-monitoring sidecar, auto-cordon and auto-rotate (`mod-106`).

### Toil budget, applied

A **toil budget** is an explicit per-rotation cap — e.g., "the primary on-call may spend at most 50% of shift hours on toil; anything over triggers a toil-reduction sprint in the next cycle." Wire this to your on-call-health dashboard (chapter 03) so the signal is visible at the SLO review.

## 4. Embedded vs. consulting SRE — rebased

Two team shapes from [SRE Workbook, Part II — Practices](https://sre.google/workbook/).

- **Embedded SRE.** One or more SREs sit inside a product / platform team for a defined engagement (3–12 months). They own the reliability work directly, with the eventual exit criterion of leaving the team production-ready and self-sufficient.
- **Consulting SRE.** The SRE team is organizationally separate, providing advice, review, SLO infrastructure, shared tooling, and on-call support — but *not* day-to-day engineering inside any one product team.

### Which shape fits an AI SRE team at level 35

In early 2024 most AI-reliability work started embedded — a classical SRE parachuted into a product team shipping a new LLM feature, or a product engineer growing into reliability work on the same team. The embedded shape works when (a) there is a single AI surface and (b) the reliability craft and the product craft sit in the same two or three engineers.

The consulting shape emerges once an org has **more than one AI product** sharing **a common serving platform** — at that point the reliability work has leverage across products and belongs with the platform team, not inside any one product. The AI SRE team at level 35 in this track's charter is the *consulting* shape: it owns the platform's reliability contract, is organizationally separate from the product teams consuming the platform, and runs shared on-call / SLO / incident infrastructure.

A good diagnostic: if your team is paged for a bug in a specific product's prompt template, your shape is wrong — either you are still embedded, or your reliability charter (`mod-101`) has not been enforced.

## 5. Launch coordination engineering — rebased

[SRE Book, Chapter 27 — Launch Coordination Engineering](https://sre.google/sre-book/launch-coordination-engineering/) describes a dedicated role (LCE) that runs a launch checklist against every new service going into production — capacity, monitoring, change management, DR, cost model, release plan, failure modes.

### What rebases for AI workloads

The checklist shape is intact. The AI-specific items added to it are:

- **Model artifact readiness** — exact model build + revision pinned, model card published, resource profile (GPU shape, context window, memory footprint) documented.
- **Prompt / tool-definition snapshot** — versioned, under the release-gate pipeline, with rollback (`mod-112`).
- **Retrieval-index snapshot pin** — the index snapshot the eval pass was run against is the snapshot that goes to prod, with the version recorded (`mod-112`).
- **Eval-set readiness** — golden stimulus set exists, judge model + version pinned, pass-rate SLO set, release gate wired (`mod-103` + eval peer).
- **Guardrail-sidecar readiness** — PII / toxicity / prompt-injection sidecars active in the critical path, with their own SLO and failure-open / failure-closed decision documented (`mod-110`).
- **Provider-fallback contract** — if the surface calls a model provider, the LiteLLM / Portkey fallback route is configured, the fallback's quality bound is known, the status-page ingestion is wired (`mod-107`).
- **Capacity sign-off** — GPU shape, pool, quota projection, HPA metric choice, Karpenter policy, scheduler-backpressure plan all in writing (`mod-106`).
- **Chaos / DR readiness** — at least one engine-restart experiment, one provider-degraded experiment, and one GPU-fault experiment run against staging with results captured (`mod-109` + `mod-113`).

Exercise 05 of this module asks you to drive a full non-abstract design review against this list. The checklist itself is operationalized in `mod-111`.

## 6. Non-abstract large-system design review — rebased

[SRE Workbook, Chapter 12 — Introducing Non-Abstract Large System Design](https://sre.google/workbook/non-abstract-design/) is the discipline of forcing a design document into *concrete* numbers before any code is written. Back-of-envelope the traffic, the per-request CPU / RAM / network, the fleet size, the failure domains, the cost. "Abstract" plans survive review and fail in production; non-abstract plans surface their failure modes on the whiteboard.

### What rebases for AI workloads

The AI-side quantities that have to land in the back-of-envelope:

- **Token throughput, not RPS.** The relevant unit of load is input + output tokens/sec aggregated across the fleet, not request rate. A 10-RPS workload with 32k-token prompts is a different platform than a 1000-RPS workload with 50-token prompts.
- **KV-cache memory footprint.** `bytes_per_token × max_concurrent_sequences × average_context_length × num_layers × hidden_dim × 2 (K + V)`. A single H100 running a 70B model has a finite pool; you cannot admit more concurrent sequences than the cache fits. Covered in `mod-106`.
- **Prefill vs. decode compute mix.** Prefill is compute-bound and FLOP-dominated; decode is memory-bandwidth bound. A workload heavy on long prompts (summarization, RAG) has different headroom than a workload heavy on long generations (coding, agent loops). The disaggregated-serving option (NVIDIA Dynamo — chapter 06) is designed around this split.
- **Provider quota pool.** For hybrid deployments, the per-provider RPM / TPM / per-model quota is a bounded resource; the design has to budget against it.
- **Cost envelope per 1M successful tokens.** Classical designs budget dollars per month; AI designs budget dollars per 1M successful tokens and derive the monthly ceiling from the projected volume.

Exercise 05 asks you to drive a non-abstract design review against a named AI stack and produce the artifact (design doc + review minutes + residual-risks list).

## Summary

- **SLI / SLO / error budget** — same abstraction; event granularity, good-event definition, and valid-event definition all re-authored for AI. Multi-dimensional SLO stack (availability + latency + quality + cost) is the AI-shaped form.
- **MTTR / MTBF / MTTA / MTTD** — same measurements; report *stratified by failure class* for AI, not as single aggregates. Add prompt / config / provider revert paths to the MTTR accounting.
- **Toil** — same seven criteria from the SRE Book; the AI surface generates AI-specific toil (canary analysis, quota tickets, drift review, model promotion, prompt hotfix, GPU reboot) that has to be automated to the same standard.
- **Embedded vs. consulting** — this track's level-35 AI SRE team is a *consulting* shape against product teams, not embedded in them.
- **Launch coordination engineering** — same checklist shape; AI-specific items added (model artifact, prompt / tool-def, retrieval index, eval set, guardrails, provider fallback, GPU capacity, chaos readiness).
- **Non-abstract design review** — same discipline; the quantities to back-of-envelope are AI-specific (token throughput, KV-cache footprint, prefill vs. decode mix, provider quota, cost per 1M tokens).

The next two chapters apply this vocabulary: chapter 02 authors an error-budget policy for a non-deterministic system; chapter 03 designs an on-call rotation against that budget.
