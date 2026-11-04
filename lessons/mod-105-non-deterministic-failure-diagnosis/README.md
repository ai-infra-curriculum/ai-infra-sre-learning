# mod-105-non-deterministic-failure-diagnosis: Non-Deterministic Failures — Diagnosing AI-Specific Incidents

**Estimated effort:** 18 hours

The observability stack from `mod-104` surfaces events. The SLO stack from `mod-103` converts events to burn. This module teaches the AI SRE the shape of the *incidents* behind the burn — the ones classical SRE did not teach, where there is no 500, no crash loop, no obvious resource exhaustion, and the only signal is that the output stopped being what the business thought it was. The module's job is **recognise and route**: name the failure, confirm the class from signals already on the dashboard, hand the diagnosis to the owning peer. Depth lives with the peer.

## Learning objectives

- Catalog the AI-specific failure classes that classical SRE does not teach — silent output drift (model-weight change, kernel-regression, numeric-format change), hallucination-rate spike, toxic / unsafe output blast, refusal-rate spike, prompt-cache cliff (prefix invalidation, template change, tokenizer change), retrieval-freshness regression (index staleness, embedding drift, document corruption), agent-tool-call cascade (recursive tool-loop, rate-limit on nested tool, mis-plumbed tool schema), judge-model drift (the judge is wrong, not the model), provider-degraded mode (silent latency regression + quality regression without an error code).
- Ground the catalog in authoritative sources — the NIST AI RMF Generative AI Profile risk taxonomy, the AI Incident Database, the OECD.AI Incidents Monitor, the MIT AI Risk Repository — so the vocabulary learners carry is defensible against regulator / auditor review.
- Build detection discipline for each class — how you notice silent drift (periodic canary-set evals, output-distribution comparison, cost / latency as proxy signals), how you notice a prompt-cache cliff (hit-rate drop + TTFT regression on the same deploy), how you notice an agent cascade (tool-call depth histogram, provider-cost blow-up).
- Build triage discipline — how to decide if the incident lives with the serving system (ai-infra-inference-engineer peer), the kernel (ai-infra-performance peer), the ML platform / registry (ai-infra-ml-platform peer), the application / prompt layer (AI Engineering family), the eval layer (ai-eval-engineer peer), or the model provider — with a decision-matrix the on-call can follow at 3 AM.
- Walk mock incidents end-to-end for the top ten failure classes — surface, diagnose, mitigate, hand off — with trace / metric / log artifacts for each.
- Cite the boundaries carefully — this module teaches how to recognise and route AI-specific incidents, not how to fix them at depth; depth lives with the owning peer.

## Chapter index

- [`01-ai-specific-failure-taxonomy.md`](./01-ai-specific-failure-taxonomy.md) — Why classical SRE's failure taxonomy is insufficient for model-backed services; the ten classes an AI SRE must be able to name without hesitation.
- [`02-grounding-in-nist-aiid-oecd-mit.md`](./02-grounding-in-nist-aiid-oecd-mit.md) — Mapping the working taxonomy to the NIST AI RMF Generative AI Profile, AI Incident Database, OECD.AI Incidents Monitor, and MIT AI Risk Repository; the defensible vocabulary.
- [`03-silent-output-drift-and-canary-evals.md`](./03-silent-output-drift-and-canary-evals.md) — Model-weight change, kernel-regression, and numeric-format change as three distinct sub-classes; canary-set evals, output-distribution comparison, and cost / latency as proxy signals.
- [`04-hallucination-toxic-refusal-spikes.md`](./04-hallucination-toxic-refusal-spikes.md) — Quality-shaped failures where the signal is a rate change, not an error; detection with judge-pipeline output, safety classifier output, and refusal counters.
- [`05-prompt-cache-cliffs-and-retrieval-regressions.md`](./05-prompt-cache-cliffs-and-retrieval-regressions.md) — Prefix invalidation, template change, tokenizer change, index staleness, embedding drift, document corruption; the two-signal joint detection that works.
- [`06-agent-tool-call-cascades.md`](./06-agent-tool-call-cascades.md) — Recursive tool loops, rate-limit on nested tools, mis-plumbed tool schemas; tool-call-depth histograms and provider-cost blow-up as the surface signals.
- [`07-judge-drift-and-provider-degraded-mode.md`](./07-judge-drift-and-provider-degraded-mode.md) — When the *measurement* is wrong, not the model; when the provider degrades silently without changing an error code.
- [`08-triage-decision-matrix-and-ten-walkthroughs.md`](./08-triage-decision-matrix-and-ten-walkthroughs.md) — A decision-matrix the on-call can follow at 3 AM; ten end-to-end mock incident walkthroughs that compose the module.

## Structure

- `01-…md` … `08-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## How this module composes with the track

- **Input from `mod-104`.** Every detection signal cited below is an OTel span attribute, a Prometheus metric, a DCGM counter, an LLM-trace platform record, a drift signal, or a structured log — all authored in `mod-104`. If the signal is not already on the dashboard, the diagnosis cannot run.
- **Input from `mod-103`.** The SLI / SLO contract is the *trigger* for every incident in this module. Burn rate fires; the on-call opens the triage matrix from chapter 08.
- **Peer boundaries from `mod-101` chapter 02 and `mod-104` chapter 07.** The triage matrix routes to those peers; this module does not duplicate their playbooks.
- **Output to `mod-108`.** The incident-response module turns the recognise-and-route discipline here into the full incident lifecycle — IC, comms, retrospective.
- **Output to `mod-109`.** The chaos module uses this taxonomy as the menu of failure modes to inject.
