# The Top-Twenty AI-Specific Runbook Library

## Why this chapter exists

Classical SRE lives in a runbook library: one document per named failure, each written so a sleep-deprived on-call can execute it at 3 AM without judgement calls. `disk-full`, `host-down`, `database-failover`, `certificate-renewal-expired`, `cert-manager-stuck`, `kafka-consumer-lag` — the shelf is stable, the shape is boring, the industry has thirty years of style guidance on how to write them.

Model-backed services have a parallel shelf, and almost none of it is written down anywhere with a name. The on-call opens the page, sees "TPOT p95 doubled since 12:40Z," and must remember what "TPOT" means, which signals rule out the obvious causes, who to loop in, and what mitigations are safe to try. Every team re-invents the same decision tree, usually during an incident, usually with a different vocabulary than the team next door.

This chapter is the first one off the shelf. It defines the runbook shape the AI SRE function owns, and names the top-twenty AI-specific runbooks the on-call must have on the day they go live with a model-backed service. The runbook authoring itself (exercise 01) is the practice; this chapter is the specification the exercise follows.

The scope is deliberate. These are *recognise-and-mitigate* runbooks, not fix-the-root-cause playbooks. The root-cause fix lives with the owning peer — the inference engineer, the ML platform engineer, the eval engineer, the application developer. The AI SRE's job in the first thirty minutes is to **detect the class, mitigate the user impact, loop in the right owner, and keep the incident contained**. `mod-105` built the recognise-and-route vocabulary; this chapter codifies the "and-mitigate" step as a durable artifact.

## 1. The runbook shape

Every runbook in the library has the same seven sections. The shape is a hard contract: the on-call must be able to open any of the twenty and find the same information in the same places. Freeform documents are not a library.

### 1.1 The header

- **Name.** A stable, grep-able slug (`rb-ttft-regression`, `rb-nccl-hang`). The slug is what the alert links to, what the post-mortem cites, what the CI test for runbook-vs-alert coverage keys on.
- **One-sentence symptom.** "TTFT p95 for route X is more than 2× its 7-day baseline for 10 consecutive minutes." If the on-call cannot match the fired alert to the symptom in five seconds, the runbook is misnamed.
- **Severity ladder.** The incident severity this class usually triggers (SEV-1 / SEV-2 / SEV-3), and the signals that escalate or de-escalate it. The ladder is not a decision — it is the default the IC may override.
- **Peer owner.** The peer function responsible for the root-cause fix (chapter 07 of this module; `mod-101` chapter 02 for the full map). The on-call pages this peer when the runbook escalates.

### 1.2 The detection signals

The specific SLI / SLO / dashboard panel / log query / trace attribute that lit this runbook up. Every signal is grounded in a `mod-104` artifact: an OTel span attribute, a Prometheus metric, a DCGM counter, an LLM-trace platform field, a structured log. A runbook that invents signals is one the on-call cannot verify.

Each signal carries:

- The metric or query name (exact — `inference_ttft_seconds` not "time to first token").
- The threshold or anomaly shape (absolute, relative, z-score, EWMA burn).
- The dashboard panel link (durable, not a Grafana ad-hoc URL).
- The expected lag between root cause and signal — because "latency took five minutes to appear" is a different problem than "latency appeared instantly."

### 1.3 The confirm checklist

The on-call's first four to six minutes. Each step is a question the on-call answers yes / no / unknown:

1. **Scope.** Which routes, which models, which regions, which deployment versions, which account tiers? A single-route event is not a fleet event.
2. **Correlation.** What else changed in the same window — a deploy, a provider status update, a routing change, a traffic shift, a cache deploy, a kernel upgrade?
3. **Baseline sanity.** Is the baseline itself still valid, or has the workload shifted?
4. **Upstream health.** What does the vendor's status page (ingested via `mod-107` chapter 04) say for the relevant routes?
5. **Downstream health.** Do the downstream consumers (business dashboards, revenue dashboards, customer-support volume) corroborate?

The checklist is deliberately short. Sprawling checklists are not read in incidents.

### 1.4 The mitigation menu

A ranked list of mitigations, each tagged with:

- **Blast radius.** User-facing / dev-facing / fleet-wide.
- **Reversibility.** One-command rollback vs. requires-coordination unwind.
- **Pre-conditions.** The signal or state that makes the mitigation safe. "Rollback is safe only when the previous version's eval score is still green."
- **Expected time-to-effect.** "Router breaker opens in 30 s;" "pod restart takes 2–3 min;" "model-swap propagates over 10 min."

The on-call picks the lowest-blast-radius, highest-reversibility option that addresses the symptom. "Try X, if it doesn't work in N minutes, try Y" is the shape. "Roll everything back immediately" is almost never the right first move for AI-specific incidents, because the state involved (model weights, vector indices, prompt caches) does not roll back the way a container image does.

### 1.5 The handoff

- **Who to page.** The peer owner for the root cause (the "on-call for X" rotation, not a named person).
- **What to send them.** The runbook's structured handoff payload — incident ID, timeline so far, confirm-checklist state, mitigations tried, next decision needed.
- **When to escalate severity.** The explicit conditions that move SEV-3 → SEV-2 or SEV-2 → SEV-1.

### 1.6 The user communication template

If the runbook class usually has user-visible impact, a one-paragraph status-page template with named placeholders. The IC and the comms owner do not draft prose in an active incident; they fill blanks.

### 1.7 The regression-fixture reminder

A one-line note: "If this runbook fires in production, chapter 05's back-feed cycle produces at least one permanent fixture. The post-mortem owner lists candidates (alert, canary case, eval entry, chaos experiment, runbook amendment) in the retro." This closes the SLO-stack loop without burying it in a separate document.

A reference row of each runbook's YAML header is given in §4 as the template.

## 2. Why these twenty

The library is a floor, not a ceiling. These twenty cover the failure surface the AI SRE will touch in the first six months on a mid-sized model-backed product: latency, cost, quality, safety, hardware, networking, model artifacts, retrieval, evals, structured output, streaming. Each runbook corresponds to a failure class named in `mod-105` or an operational surface named in `mod-103`, `mod-104`, `mod-106`, or `mod-107`. The through-line is that the class fires an SLO burn, the runbook exists, the on-call executes it, and the post-mortem produces a fixture that would have caught it earlier.

Three organising principles explain the shape:

- **Serving-path runbooks** (TTFT, TPOT, cache, preemption, streaming, structured output) sit with the inference engineer peer for root-cause fix; the AI SRE's mitigations route traffic, open breakers, and roll serving configs.
- **Quality-path runbooks** (hallucination, toxic, refusal, judge-drift, eval poisoning, embedding-drift, retrieval-staleness) sit with the eval engineer and AI engineer peers; the AI SRE's mitigations degrade capability (route off the bad model, suppress a tool, raise a refusal bar) and preserve safety until the owner lands a fix.
- **Infrastructure-path runbooks** (GPU driver / ECC / XID, NCCL hang, weight corruption, model-swap-rollback, provider-degraded, cost blow-up, tool-call cascade) sit with the inference engineer, performance engineer, ML platform engineer, FinOps, and vendor-risk peers; the AI SRE's mitigations drain hardware, swap models, open breakers, and cap traffic.

The naming matches `mod-105`'s taxonomy so that a learner moving from recognise-and-route into recognise-and-mitigate does not have to re-translate the vocabulary.

## 3. The twenty runbooks

Each entry below is the **specification** for the runbook — the symptom, the detection signals, the confirm steps, the mitigation menu, the handoff, and the back-feed note. A real runbook adds environment specifics (metric names, cluster names, routing control-plane commands). The exercise is to produce the environment-specific version for a plausible product.

### 3.1 `rb-ttft-regression` — TTFT regression

- **Symptom.** `inference_ttft_seconds` p95 on a serving route doubled from its 7-day baseline for ≥ 10 min.
- **Severity ladder.** SEV-3 by default; SEV-2 if the SLO burn-rate alert fires at the 1-h window; SEV-1 if the route is a top-5 revenue surface and the burn crosses the 5-min window.
- **Peer owner.** `ai-infra-inference-engineer`.
- **Detection signals.** `inference_ttft_seconds` histogram (per model, per route, per region); scheduler admission queue depth; prefill-stage GPU utilisation; request-arrival rate change; prompt-cache hit rate on the same route; model-swap event stream.
- **Confirm checklist.**
  1. Is the regression isolated to one model, one route, one replica group, or one zone?
  2. Did anything deploy in the window — serving image, scheduler config, prefill-batch cap, prompt template, cache policy, model weights?
  3. Did arrival rate or prompt-length distribution change (long-context queries can distort TTFT)?
  4. Is prompt-cache hit rate off its baseline on this route (chapter 01 §3.3 of `mod-105`'s cache-cliff signal)?
  5. Is the provider status page green for every upstream this route touches?
- **Mitigation menu.**
  1. **Open the router breaker** on the affected route if a fallback route exists (see `mod-107` chapter 02 for fallback contract).
  2. **Lower the prefill-batch cap** to reduce head-of-line blocking at the cost of throughput.
  3. **Shed non-critical traffic** (background / batch) via the admission policy.
  4. **Roll back** the serving image if a deploy is the most recent correlated event and eval evidence is still green on the prior version.
  5. **Pin requests** to the healthy replica group / zone.
- **Handoff.** Page `ai-infra-inference-engineer` on-call with the incident ID, symptom window, scope, correlated change, and mitigation state.
- **User communication.** "Some responses may begin more slowly than usual. We are investigating and have taken steps to reduce impact."
- **Fixture back-feed.** Alert thresholds, a load-shape case, a canary-prompt set that exercises the slow path.

### 3.2 `rb-tpot-regression` — TPOT (time per output token) regression

- **Symptom.** `inference_tpot_seconds` p95 on a serving route doubled from its 7-day baseline for ≥ 10 min.
- **Severity ladder.** SEV-3 by default; SEV-2 at 1-h burn; SEV-1 if the route is a streaming-critical chat surface and the burn crosses the 5-min window.
- **Peer owner.** `ai-infra-inference-engineer`.
- **Detection signals.** `inference_tpot_seconds`; per-replica throughput tokens / s; decode-stage GPU utilisation and SM occupancy (DCGM `DCGM_FI_PROF_SM_OCCUPANCY`); batch-size distribution at decode; KV-cache memory pressure; speculative-decoding acceptance rate (if in use).
- **Confirm checklist.**
  1. Scope — one model, one route, one replica group, one zone?
  2. Decode batch-size distribution — did it shift (towards 1, which collapses throughput, or towards the cap, which saturates)?
  3. KV-cache eviction rate — is the cache thrashing because context lengths grew?
  4. If speculative decoding is in use, did acceptance rate drop (misconfig / mismatched draft model)?
  5. Scheduler config changed?
- **Mitigation menu.**
  1. **Lower the max context cap** on the route to reduce KV pressure.
  2. **Raise or lower the batch-size target** depending on whether throughput or latency is binding.
  3. **Disable speculative decoding** if acceptance rate is anomalous.
  4. **Drain the suspect replica group** and let the scheduler re-balance.
  5. **Router fall-through** to an equivalent model with fewer parameters, if the fallback contract allows.
- **Handoff.** `ai-infra-inference-engineer` on-call, same payload shape as `rb-ttft-regression`.
- **User communication.** "Streaming responses may be slower than usual."
- **Fixture back-feed.** Load-shape that reproduces the batch-size collapse; dashboard panel for speculative-decoding acceptance rate.

### 3.3 `rb-cache-hit-cliff` — Prompt-cache hit-rate cliff

- **Symptom.** Prompt-cache hit rate on a route dropped ≥ 20 percentage points over ≤ 30 min, usually paired with a TTFT regression on the same route (`rb-ttft-regression` may also fire).
- **Severity ladder.** SEV-3 for a single route; SEV-2 if the cache is shared and multiple routes are regressed.
- **Peer owner.** `ai-infra-inference-engineer` for serving-side cache; `AI Engineering` for prompt-template changes.
- **Detection signals.** `prompt_cache_hit_ratio` (per route, per model); prefix-length histogram; template-version counter; tokenizer-version counter; template-change deploy event.
- **Confirm checklist.**
  1. Did the prompt template, system prompt, or scaffolding change in the window?
  2. Did the tokenizer or model version change (both invalidate prefix cache)?
  3. Did the request shape change (new leading field, new user-locale injection)?
  4. Is the cache memory budget full (eviction thrash)?
  5. Did a new tenant or workload start sharing the cache?
- **Mitigation menu.**
  1. **Roll back the template change** if it correlates with the cliff and the prior template is still green on evals.
  2. **Pin the template version** on the route until the application team lands a fix that preserves the stable prefix.
  3. **Raise the cache memory budget** or shard the cache per route if a shared cache is being contended.
  4. **Accept the cost / latency** for the window and route-shift to a vendor route with its own cache (see `mod-107` chapter 02).
- **Handoff.** `ai-infra-inference-engineer` and the application team that owns the template.
- **User communication.** Usually silent (the regression shows up as slower responses, covered by `rb-ttft-regression`'s template).
- **Fixture back-feed.** A canary that tests a fixed stable prefix against the cache metric; a CI check that blocks template changes that move the prefix hash.

### 3.4 `rb-preemption-spike` — GPU preemption / eviction spike

- **Symptom.** Preempted or evicted serving pods exceed the 7-day baseline by ≥ 2× over 15 min, with observable TTFT / availability impact.
- **Severity ladder.** SEV-3 by default; SEV-2 if availability SLO is burning.
- **Peer owner.** `ai-infra-performance` for cluster / scheduler; `ai-infra-inference-engineer` for serving.
- **Detection signals.** Pod eviction / restart counters (Kubernetes event stream); scheduler priority-class usage; preemption decision logs; batch / training job starts in the same cluster; spot-instance reclamation signals (if applicable).
- **Confirm checklist.**
  1. Did a higher-priority workload (training, batch) start?
  2. Spot-instance reclamation event from the cloud provider?
  3. Did node-pool capacity shrink (autoscaler scale-down, cordon, drain)?
  4. Is the cluster running at or near capacity for GPU resources?
  5. Were any priority classes or PodDisruptionBudgets modified recently?
- **Mitigation menu.**
  1. **Raise the PDB** on serving workloads if it is lower than necessary.
  2. **Pause the lower-priority workload** (batch / training) briefly if its SLO allows.
  3. **Add capacity** from reserved pool, preferring same-zone replacement.
  4. **Shift traffic** to an unaffected region if cross-region is in the fallback contract.
  5. **Switch to a non-spot node pool** for the serving workload until the volatility passes.
- **Handoff.** `ai-infra-performance` for scheduler decisions; `ai-infra-inference-engineer` if serving-config change is required.
- **User communication.** Often silent if mitigation is fast; template exists for sustained impact.
- **Fixture back-feed.** Chaos experiment that preempts a serving pod and validates the recovery time; alert on PDB / priority-class misconfig.

### 3.5 `rb-hallucination-rate-spike` — Hallucination-rate spike

- **Symptom.** Judge-pipeline output or groundedness-metric judge score on a route moved ≥ N standard deviations worse from its baseline for ≥ 1 h, with no corresponding product-change event. Grounded in `mod-105` chapter 04.
- **Severity ladder.** SEV-2 by default; SEV-1 if the route touches a regulated domain (health, finance, legal) or if customer-facing content is being published.
- **Peer owner.** `ai-eval-engineer` and `AI Engineering`.
- **Detection signals.** Judge-pipeline score (hallucination / groundedness / faithfulness); retrieval hit rate and recall@k (for RAG routes); refusal rate (inversely correlated); customer-support ticket volume on "incorrect answer" tags.
- **Confirm checklist.**
  1. Did the model, model-version, or provider route change in the window (`mod-105` chapter 03)?
  2. Did retrieval change — index refresh, embedding model, chunking, filter?
  3. Did the prompt / system prompt change?
  4. Is the judge itself behaving normally (`rb-judge-model-drift`)? If the judge is wrong, nothing else is interpretable.
  5. Did input distribution shift (new user cohort, new workload type)?
- **Mitigation menu.**
  1. **Route off the offending model** (back to the previous-known-good via the fallback contract).
  2. **Roll back the retrieval change** if a refresh / chunking change correlates.
  3. **Roll back the prompt** if a template deploy correlates.
  4. **Raise the refusal / uncertainty threshold** to trade coverage for safety.
  5. **Suspend auto-publish** workflows that use the output unmoderated.
- **Handoff.** `ai-eval-engineer` for diagnosis; `AI Engineering` for prompt / retrieval ownership; `ai-risk-engineer` + legal if it may trigger compliance reporting (chapter 06 of this module).
- **User communication.** "We have observed a quality regression in [feature]. The feature is temporarily reduced in capability while we investigate." Avoid vendor naming.
- **Fixture back-feed.** New canary eval case that reproduces the regression; dashboard panel added for the judge metric on this route; threshold-based alert added to the SLO stack.

### 3.6 `rb-toxic-output-blast` — Toxic / unsafe output blast

- **Symptom.** Safety-classifier output rate exceeds its 7-day baseline by ≥ 3× over 15 min, or any single egregious event is detected by any downstream flow (user report, support, automated screenshot).
- **Severity ladder.** SEV-1 by default. User-safety incidents are escalated on first report, not on burn aggregation.
- **Peer owner.** `AI Engineering` and `ai-risk-engineer`.
- **Detection signals.** Safety-classifier output (toxicity, self-harm, hate, sexual, violence categories); flagged-output event log; rate of user-reported safety flags; news / social monitoring signal (if tooled).
- **Confirm checklist.**
  1. Is the signal real or a classifier false-positive spike (check the samples before paging the executive).
  2. Scope — fleet-wide, one model, one route, one cohort, one locale?
  3. Did the prompt / scaffolding change?
  4. Did the moderation path change (new filter version, disabled filter, mis-wired filter)?
  5. Did the model route change silently (vendor-side)?
- **Mitigation menu.**
  1. **Raise moderation strictness / reinstate the filter** immediately.
  2. **Switch to a safer route** (lower-capability / stricter-aligned model) via the fallback contract.
  3. **Halt the specific feature** that is emitting the content (feature flag).
  4. **Halt auto-publish / auto-send** to any external surface (email, social, support).
  5. **Collect samples** for the safety team under the retention policy — do not dispose.
- **Handoff.** `ai-risk-engineer`, `AI Engineering`, legal, PR, exec on-call. Chapter 06 of this module for the formal-notification path.
- **User communication.** Legal / PR owns the externally visible statement. The SRE's job is to brief with facts, not to draft the statement.
- **Fixture back-feed.** New classifier threshold; new eval case; chaos experiment that disables the safety filter in a canary and validates detection; post-mortem co-authored with safety.

### 3.7 `rb-refusal-rate-spike` — Refusal-rate spike

- **Symptom.** Model refusal rate on a route exceeds the 7-day baseline by ≥ 3× over 1 h, or a specific legitimate category is uniformly refused.
- **Severity ladder.** SEV-3 by default; SEV-2 if customer-impact is specific (legal / medical / safety advisory refused on grounded queries); SEV-1 if the refusal is masking a toxic-output regression.
- **Peer owner.** `AI Engineering` and `ai-eval-engineer`.
- **Detection signals.** Refusal counter (keyword-based plus judge-based); refusal-by-category histogram; quality-score correlated with refusal; model-version event.
- **Confirm checklist.**
  1. Did the model or provider route change?
  2. Did the system prompt change (over-cautious scaffolding)?
  3. Did the safety filter change (over-blocking)?
  4. Is the refusal uniform or specific to a category / cohort / locale?
  5. Is customer-support reporting correlated complaints?
- **Mitigation menu.**
  1. **Roll back the prompt** if a scaffolding change correlates.
  2. **Route off the model** if provider-side alignment change is the likely cause.
  3. **Loosen the safety filter** only with eval evidence that toxic output is not rising concurrently.
  4. **Communicate the reduced capability** honestly if resolution will take hours.
- **Handoff.** `AI Engineering` for prompt; `ai-eval-engineer` for evaluating safety-utility trade-off; `ai-risk-engineer` if alignment change is vendor-side.
- **User communication.** "[Feature] is being more cautious than usual while we tune. Thank you for your patience." Avoid defensive language.
- **Fixture back-feed.** Eval set of legitimate queries this route must answer; threshold alert on refusal rate.

### 3.8 `rb-tool-call-cascade` — Agent tool-call cascade

- **Symptom.** Tool-call depth histogram p99 moves beyond the 7-day baseline, or per-session tool-call count exceeds a cap, or per-session provider cost exceeds a cap. Grounded in `mod-105` chapter 06.
- **Severity ladder.** SEV-2 by default (cost and latency surface); SEV-1 if the cascade touches an external side-effect (payment, publish, send) in a loop.
- **Peer owner.** `AI Engineering`.
- **Detection signals.** `agent_tool_call_depth` histogram; per-session token / cost burn; per-session tool-call count; tool-rate-limit 429 counter; recursive-tool-call detector (same tool-name chain).
- **Confirm checklist.**
  1. Did the agent scaffolding, tool schema, or tool set change?
  2. Is one tool returning an error shape the agent retries into?
  3. Is the orchestrator honouring the max-depth / max-cost cap?
  4. Are cost and token metrics in the alert window congruent with actual billing (chapter `mod-107` for cross-check)?
  5. Did model-version change the planner behaviour?
- **Mitigation menu.**
  1. **Lower the agent's max-tool-call cap** per session.
  2. **Rate-limit** the specific tool if it is in the loop.
  3. **Disable the specific tool** via feature flag if its failure drives the loop.
  4. **Route off the model** if the planner behaviour correlates with a model-version change.
  5. **Suspend agentic sessions** entirely for the affected cohort if side-effects are at risk.
- **Handoff.** `AI Engineering` for agent scaffolding; `ai-infra-inference-engineer` if routing is mis-plumbed; FinOps if cost is the user-visible impact.
- **User communication.** If sessions stall, acknowledge; otherwise usually silent.
- **Fixture back-feed.** Chaos experiment that returns a looping error shape from a tool and asserts the cap holds; alert on tool-call depth and per-session cost.

### 3.9 `rb-provider-degraded-fallback` — Provider degraded, fall through

- **Symptom.** Vendor status-page event (`mod-107` chapter 04) or local-breaker open signal on a provider route for ≥ 5 min, with user-visible availability or latency impact on dependent routes.
- **Severity ladder.** SEV-3 by default; SEV-2 if multiple routes are impacted; SEV-1 if the fallback is not covering (chapter 02 `mod-107`).
- **Peer owner.** `ai-infra-inference-engineer` for router; `ai-risk-engineer` for vendor communication.
- **Detection signals.** Status-page ingestor event; per-provider error-rate and latency (`mod-104`); fallback-route traffic ratio; breaker state (open / half / closed).
- **Confirm checklist.**
  1. Is the status event scoped to the model / region we depend on?
  2. Is the router actually routing away (traffic ratio on the fallback route is growing)?
  3. Is the fallback healthy — not silently degraded in the same window?
  4. Did the equivalence table change recently (eval score drop on the fallback)?
  5. Does the residency / compliance contract permit the fallback for this cohort?
- **Mitigation menu.**
  1. **Confirm the breaker is open** (manually force-open if the automatic signal is lagging).
  2. **Shift traffic** per the fallback contract; monitor the fallback for secondary degradation.
  3. **Degrade capability** (lower-cost / lower-capability model) on the fallback if primary cannot be reached and fallback capacity is tight.
  4. **Communicate to customers** who have a per-route commitment (residency / data-zone).
- **Handoff.** `ai-infra-inference-engineer`; `ai-risk-engineer` for any multi-hour event (chapter 06 for formal-notification trigger).
- **User communication.** "An upstream provider is degraded. We have routed traffic to an alternate provider; some features may be temporarily reduced." Avoid naming the vendor unless they have acknowledged publicly first.
- **Fixture back-feed.** Chaos experiment that opens the breaker and asserts full fallback; dashboard panel for per-provider error-rate.

### 3.10 `rb-gpu-driver-ecc-xid` — GPU driver / ECC / XID incident

- **Symptom.** DCGM reports XID errors, ECC double-bit errors, or driver watchdog events on one or more GPUs; serving replicas on those hosts error or stall.
- **Severity ladder.** SEV-3 for isolated single-host; SEV-2 if multiple hosts or a generation-wide driver issue; SEV-1 if availability SLO burns.
- **Peer owner.** `ai-infra-performance` and `ai-infra-platform`.
- **Detection signals.** DCGM `DCGM_FI_DEV_XID_ERRORS`, ECC counters, NVML events, driver watchdog logs, Kubernetes node condition `NodeNotReady` with GPU-labelled taint, pod-restart loop on GPU hosts.
- **Confirm checklist.**
  1. Which XID codes — some are benign (process termination), some are fatal (uncorrectable ECC). Reference NVIDIA's XID documentation.
  2. Scope — one host, one rack, one zone, one GPU generation?
  3. Was there a driver / CUDA / NCCL upgrade in the window?
  4. Is there a correlation with specific workloads (one model, one batch size)?
  5. Does the fleet have a known fleet-wide driver advisory?
- **Mitigation menu.**
  1. **Cordon and drain** the affected node(s).
  2. **Reboot** the host if the error class allows (per NVIDIA guidance).
  3. **Roll back the driver** on a canary host if a driver-upgrade correlates.
  4. **Shift the replica set** away from the affected generation if the issue is generation-wide.
  5. **Open a vendor support ticket** with DCGM capture attached if the error is uncorrectable.
- **Handoff.** `ai-infra-performance`, `ai-infra-platform`, vendor TAM for hardware RMA / driver bulletins.
- **User communication.** Usually silent unless capacity loss is sustained.
- **Fixture back-feed.** Chaos experiment that injects an XID in staging; alert on XID counters; runbook step added for known XID codes.

### 3.11 `rb-nccl-hang-straggler` — NCCL hang / slow-rank straggler

- **Symptom.** Collective (training or sharded-inference) stalls or runs below throughput baseline; one rank consistently lags in step time; NCCL watchdog or timeout fires.
- **Severity ladder.** SEV-3 if a training job hangs; SEV-2 if sharded inference capacity is lost; SEV-1 if both.
- **Peer owner.** `ai-infra-performance` (networking / NCCL), `ai-infra-platform` (fabric).
- **Detection signals.** NCCL log output (`NCCL_DEBUG=INFO`), per-rank step-time distribution, InfiniBand / RoCE link counters, GPU utilisation gap across ranks, NCCL watchdog timeout.
- **Confirm checklist.**
  1. Is one rank consistently slow, or is the stall symmetric (a hang)?
  2. Did the topology change (new nodes, NIC firmware, switch config, routing)?
  3. Is there a known-bad host in the allocation?
  4. Were NCCL env vars or algorithm hints changed?
  5. Is the fabric showing CRC errors / port flaps on the affected links?
- **Mitigation menu.**
  1. **Eject the slow rank's host** from the allocation; relaunch the job.
  2. **Test NCCL with `nccl-tests`** on the suspect hosts after cordoning.
  3. **Revert NCCL env / algorithm** changes.
  4. **Request a different allocation** (scheduler re-pick) to avoid the known-bad switch.
- **Handoff.** `ai-infra-performance` for NCCL / fabric diagnosis.
- **User communication.** Usually silent for training; drained-capacity template for inference.
- **Fixture back-feed.** Pre-job NCCL health-check; alert on per-rank step-time skew.

### 3.12 `rb-weight-corruption` — Model-weight corruption

- **Symptom.** Loading a model fails checksum / signature verification, or inference produces nonsensical output (NaN loss, all-tokens-identical output, output quality collapse immediately after a weight load).
- **Severity ladder.** SEV-1 by default. Serving corrupt weights at user-facing scale is a safety-adjacent incident.
- **Peer owner.** `ai-infra-ml-platform` (registry), `ai-infra-inference-engineer` (serving).
- **Detection signals.** Checksum / signature verification failure at load; load-time logs; post-load canary eval failure; NaN / Inf detectors; sudden quality-metric drop pinned to a weight-swap event.
- **Confirm checklist.**
  1. Is checksum verification configured and passing? If disabled, that is the first fix.
  2. Did the registry report a successful checksum at publish time?
  3. Which version is loaded, and does it match the pinned config?
  4. Is the issue reproducible on a second host with the same artifact?
  5. Does the artifact storage (S3, GCS, Azure Blob) show any prior upload failure or partial-copy event?
- **Mitigation menu.**
  1. **Refuse to serve** the suspect version at the gateway (fail closed).
  2. **Roll back** to the last known-good version pinned in the registry.
  3. **Quarantine** the suspect artifact pending root-cause.
  4. **Force re-download** on all replicas to rule out a per-host corruption vs. a registry-level corruption.
- **Handoff.** `ai-infra-ml-platform` immediately. If customer-visible output was possibly affected, loop in `ai-risk-engineer` and legal (chapter 06).
- **User communication.** If user-visible, the template emphasises the capability regression and the active rollback; do not describe the artifact details publicly.
- **Fixture back-feed.** Checksum-on-load enforcement; post-load canary eval gate; alert on NaN / Inf; registry integrity audit cadence.

### 3.13 `rb-model-swap-rollback` — Model-swap-rollback

- **Symptom.** A model deploy / swap landed and a quality, latency, or cost metric regressed beyond threshold; rollback is required.
- **Severity ladder.** SEV-3 by default; SEV-2 or SEV-1 depending on the regressed metric.
- **Peer owner.** `ai-infra-ml-platform`, `ai-infra-inference-engineer`, `ai-eval-engineer` for eval decision.
- **Detection signals.** Eval-set canary score on the new version; progressive-rollout traffic fraction; per-version SLI panels; cost per request on the new version vs. old.
- **Confirm checklist.**
  1. Is the regression statistically significant on the canary eval (not within-noise)?
  2. Is the previous version still present and loadable?
  3. Was the swap atomic, or is part of the fleet still on the old version?
  4. Is the regression quality, latency, cost, or safety — each class has different urgency.
  5. Is a forward-fix viable in the window (new weight, config patch), or is rollback the right call?
- **Mitigation menu.**
  1. **Freeze the rollout** at the current traffic fraction.
  2. **Reverse the traffic fraction** back to the prior version.
  3. **Unload the new version** from the replica fleet if the swap was replica-local.
  4. **Communicate** the rollback to the model owner; do not silently revert.
- **Handoff.** `ai-infra-ml-platform` owns the registry decision. `ai-eval-engineer` signs off on the metric interpretation.
- **User communication.** Usually silent for successful rollback.
- **Fixture back-feed.** Pre-promotion canary eval gate; progressive-rollout observability; rollback-drill cadence.

### 3.14 `rb-retrieval-index-staleness` — Retrieval-index staleness

- **Symptom.** RAG-pipeline groundedness score drops, or user-visible "my recent document isn't searchable" reports; recall@k on a canary query set drops. Grounded in `mod-105` chapter 05.
- **Severity ladder.** SEV-3 by default; SEV-2 if the freshness commitment is contractual.
- **Peer owner.** `ai-infra-pipeline` and `AI Engineering`.
- **Detection signals.** Index-ingest lag counter (time since last successful commit); last-success timestamp per source; canary-query recall@k; per-document ingest success / failure counters.
- **Confirm checklist.**
  1. Is ingest healthy or stalled (lag counter trend)?
  2. Did the ingest pipeline deploy or config change?
  3. Did the embedding model or tokenizer change (invalidates all prior embeddings)?
  4. Are failures scoped to one source / one tenant / one document class?
  5. Is the index replica set in sync (if sharded / replicated)?
- **Mitigation menu.**
  1. **Pause the ingest** if it is actively corrupting the index.
  2. **Fail over** to a replica index that is still fresh.
  3. **Rebuild the index** from the source of truth if corruption is suspected (long-running, so communicate).
  4. **Route around** the stale slice at the application layer if the taxonomy allows (per-source filter).
- **Handoff.** `ai-infra-pipeline` for ingest; `AI Engineering` for retrieval.
- **User communication.** "Search may not include documents added in the last [N] hours."
- **Fixture back-feed.** Ingest-lag alert at a tight SLO; chaos experiment that stalls a source and verifies the lag alert.

### 3.15 `rb-embedding-drift` — Embedding drift

- **Symptom.** Retrieval quality on previously-performant canary queries degrades without a visible deploy; vector-centroid or distribution-shift metric on new embeddings moves beyond tolerance. Grounded in `mod-105` chapter 05.
- **Severity ladder.** SEV-3 by default.
- **Peer owner.** `ai-infra-pipeline` and `ai-eval-engineer`.
- **Detection signals.** Embedding-model version counter; vector-space distribution-shift metric (centroid shift, MMD, KL on sampled queries); canary recall@k; new-document embedding vs. old-document embedding similarity on controlled pairs.
- **Confirm checklist.**
  1. Did the embedding-model version change (provider-side or ours)?
  2. Did the chunking or preprocessing change?
  3. Did the tokenizer change?
  4. Did the source data distribution change (new tenant, new language, new document type)?
  5. Is the drift new-document-only or does it affect old embeddings too?
- **Mitigation menu.**
  1. **Pin the embedding-model version** if a silent provider update is suspected.
  2. **Re-embed** affected documents in the background (long-running).
  3. **Route new ingests to a parallel index** on the new embedding until re-embed completes.
  4. **Communicate the degradation** to downstream consumers.
- **Handoff.** `ai-infra-pipeline`; `ai-eval-engineer` for drift measurement; `ai-risk-engineer` if the embedding provider changed silently.
- **User communication.** Typically silent beyond the general retrieval-quality template.
- **Fixture back-feed.** Alert on embedding-model-version change; canary query set with recall@k thresholds; distribution-shift metric added to the dashboard.

### 3.16 `rb-judge-model-drift` — Judge-model drift

- **Symptom.** The judge model's scoring distribution on a fixed calibration set moves beyond tolerance, invalidating any eval-derived alert built on top of it. Grounded in `mod-105` chapter 07.
- **Severity ladder.** SEV-3 by default; SEV-2 if judge-derived alerts are gating production decisions.
- **Peer owner.** `ai-eval-engineer`.
- **Detection signals.** Judge model-version counter; judge score on the fixed calibration set; judge agreement with a second judge model; human-label correlation on sampled traces.
- **Confirm checklist.**
  1. Did the judge model version change (ours or the provider's)?
  2. Did the judge prompt change?
  3. Is the shift directional (systematically higher / lower) or variance (noisier)?
  4. Is the drift large enough to invalidate the thresholds that depend on it?
  5. Are any production-gating decisions currently reading the drifted judge?
- **Mitigation menu.**
  1. **Suspend judge-derived gates** that cannot be re-calibrated in time.
  2. **Pin the judge version**.
  3. **Recalibrate thresholds** against the fixed calibration set.
  4. **Communicate** to consumers (eval owner, model owner) that eval signals are temporarily paused.
- **Handoff.** `ai-eval-engineer`.
- **User communication.** Silent (internal signal).
- **Fixture back-feed.** Fixed calibration set as a permanent artifact; alert on judge-version change; CI gate that blocks judge updates without re-calibration evidence.

### 3.17 `rb-eval-set-poisoning` — Eval-set poisoning

- **Symptom.** Eval score moves on a dataset without a corresponding model change, or eval contamination is detected (e.g., training data overlap check flags a bulk match).
- **Severity ladder.** SEV-2 by default; SEV-1 if the eval gates a regulated-domain decision.
- **Peer owner.** `ai-eval-engineer` and `ai-risk-engineer`.
- **Detection signals.** Eval-dataset checksum / version counter; contamination detector (overlap with training-data fingerprints); eval-score history; per-item score anomaly.
- **Confirm checklist.**
  1. Did the eval dataset version change? Is the checksum pinned?
  2. Is the drift uniform across items or clustered (points to contamination of specific items)?
  3. Was training data recently refreshed? Did the dataset boundary change?
  4. Are there any ingress points for user-submitted eval items?
  5. Who had write access to the eval dataset in the window?
- **Mitigation menu.**
  1. **Freeze the eval dataset** at the last known-good version.
  2. **Quarantine** the suspect items.
  3. **Suspend promotion decisions** that depend on the affected eval.
  4. **Investigate** write-access logs.
- **Handoff.** `ai-eval-engineer`; `ai-risk-engineer` if the poisoning suggests malicious action; security if write-access is compromised.
- **User communication.** Silent.
- **Fixture back-feed.** Eval-dataset checksum enforcement in CI; access-audit log alerting; periodic contamination scan.

### 3.18 `rb-structured-output-break` — Structured-output conformance break

- **Symptom.** JSON / tool-call / grammar-constrained outputs on a route fail schema validation or structured-output validator at a rate above the 7-day baseline by ≥ 3×.
- **Severity ladder.** SEV-3 by default; SEV-2 if the downstream pipeline cannot degrade gracefully.
- **Peer owner.** `AI Engineering` and `ai-infra-inference-engineer`.
- **Detection signals.** Structured-output-validator error rate; schema-mismatch counters; downstream pipeline error rate (consumer of the structured output); vendor structured-output API error class, if used.
- **Confirm checklist.**
  1. Did the schema change?
  2. Did the model / provider change?
  3. Did the tokenizer change (affects grammar-constrained decoding)?
  4. Is the vendor-side structured-output feature regressed (`rb-provider-degraded-fallback`)?
  5. Is prompt scaffolding still enforcing the schema explicitly?
- **Mitigation menu.**
  1. **Re-enable prompt-level schema reinforcement** if it was removed when vendor structured-output was turned on.
  2. **Route off the vendor's structured-output mode** back to prompt+validator while the vendor side is confirmed.
  3. **Roll back** the schema change if a schema deploy is the correlated event.
  4. **Enable the retry-with-repair** path (if the system has one; otherwise an eval gate blocks turning it on).
- **Handoff.** `AI Engineering` for schema; `ai-infra-inference-engineer` if vendor-side mode is broken.
- **User communication.** Silent in most pipelines; explicit if downstream user flows fail (template).
- **Fixture back-feed.** Schema-conformance eval; alert on validator error-rate; CI gate that blocks schema changes without downstream migration.

### 3.19 `rb-streaming-stall` — Streaming stall

- **Symptom.** Token streaming pauses longer than the SLO threshold (p99 inter-token time exceeds N × baseline) despite a healthy TPOT metric, or the stream closes early without a terminal event.
- **Severity ladder.** SEV-3 by default; SEV-2 if closed-early events are user-visible at scale.
- **Peer owner.** `ai-infra-inference-engineer` and the serving gateway owner.
- **Detection signals.** Per-stream inter-token-time distribution; connection-close reason counters; proxy / CDN layer timeout counters; buffering-vs-streaming configuration counters; end-to-end streaming duration.
- **Confirm checklist.**
  1. Is the stall at the model layer (TPOT bad) or at the gateway (TPOT fine, inter-arrival bad)?
  2. Is a proxy / CDN / load balancer buffering (disables chunked streaming)?
  3. Are keep-alive and idle timeouts configured for long streams?
  4. Did the gateway or proxy config change?
  5. Is the client disconnecting early (user-side), not the server?
- **Mitigation menu.**
  1. **Disable proxy buffering** on the streaming route.
  2. **Raise idle / keep-alive timeouts** to cover the longest expected stream.
  3. **Fail over** to a region without the misconfigured proxy.
  4. **Communicate** with affected clients if SDK behaviour is implicated.
- **Handoff.** Serving gateway owner; `ai-infra-inference-engineer`.
- **User communication.** "Some long responses may pause or end early." Template exists for the duration.
- **Fixture back-feed.** Long-stream canary; alert on inter-token-time distribution; gateway-config lint that blocks buffering on streaming routes.

### 3.20 `rb-cost-blow-up` — Cost blow-up

- **Symptom.** Hourly provider spend on a route exceeds the 7-day baseline by ≥ 3× (or whatever the per-route budget alert names) without a corresponding business-volume increase.
- **Severity ladder.** SEV-3 by default; SEV-2 if the spend exceeds the budgeted monthly cap for the route in-day.
- **Peer owner.** `FinOps` and `AI Engineering` for feature-level cause; `ai-infra-inference-engineer` for routing.
- **Detection signals.** Per-route cost per hour; tokens per request distribution; per-request cost attribution; per-tenant cost; vendor-billing export (`mod-107` cross-check); agent-session cost (shares the `rb-tool-call-cascade` detector).
- **Confirm checklist.**
  1. Did a feature launch?
  2. Did the model route change (more expensive model)?
  3. Did prompt or response length distribution shift?
  4. Is an agent cascade active (`rb-tool-call-cascade`)?
  5. Is a tenant abusing the service (abuse / scraping)?
  6. Is the vendor billing anomaly real (correlate with their ledger)?
- **Mitigation menu.**
  1. **Enable per-tenant rate caps** if abuse is suspected.
  2. **Route to a cheaper equivalent model** if the fallback contract allows.
  3. **Cap max tokens** on the suspect route.
  4. **Disable the feature flag** that correlates.
  5. **Open a vendor billing ticket** if data is suspect.
- **Handoff.** `FinOps` for the budget decision; `AI Engineering` for feature change; `ai-infra-inference-engineer` for routing; `ai-risk-engineer` if a vendor-contract clause is at stake.
- **User communication.** Internal first; user-facing only if capability is reduced.
- **Fixture back-feed.** Per-route hourly spend alert; per-tenant rate-cap gate; per-session max-cost cap.

## 4. The runbook header template

The first-line header of every runbook is YAML so the on-call tooling can parse it. The exercise's deliverable uses this shape.

```yaml
---
runbook_id: rb-ttft-regression
title: TTFT regression
symptom: inference_ttft_seconds p95 on route X doubled vs. 7-day baseline for ≥ 10 min
severity_default: SEV-3
severity_ladder:
  sev2: 1-h burn-rate alert fires on an SLO-committed route
  sev1: 5-min burn-rate alert fires on a tier-1 revenue route
peer_owner: ai-infra-inference-engineer
related_runbooks:
  - rb-cache-hit-cliff
  - rb-preemption-spike
  - rb-provider-degraded-fallback
mod_105_class: silent-latency-regression
fixture_backfeed: [alert, load-shape, canary-prompt-set]
last_reviewed: 2026-10-01
---
```

The CI test asserts that every alert that fires has a `runbook_id` field linking to one of these documents, and that every `runbook_id` resolves to a file in the library. Breaking either breaks the merge.

## 5. Keeping the library honest

Runbooks rot faster than most on-call documents because the surfaces they describe move. Three practices keep the library alive:

- **Review cadence.** Every runbook has a `last_reviewed` date. Anything older than six months is flagged red in the dashboard. The AI SRE function schedules a monthly rolling review so no runbook goes stale.
- **Incident back-feed.** Every post-mortem (chapter 03 of this module) explicitly names one or more runbook updates. "No runbook amendment" is a valid outcome, but it must be written.
- **Chaos verification.** The chaos experiments in `mod-109` exercise each runbook at least once per quarter. A runbook that no one can execute against a staged incident in under 30 minutes is a broken runbook.

## 6. The AI SRE's library-authoring worksheet

1. Fit the runbook to the seven-section shape in §1. Deviate and the on-call cannot navigate.
2. Pin each detection signal to a `mod-104` artifact. Signals that do not exist in your observability stack cannot be in the runbook.
3. Rank mitigations by blast radius and reversibility, not by cleverness. The on-call reaches for the first safe-enough option, not the optimal one.
4. Name the peer owner explicitly. A runbook without a handoff is a runbook the on-call owns forever.
5. Snapshot the YAML header. The alert pipeline reads it; CI validates it.
6. Schedule the review. The library is only honest if someone owns the calendar.

## Summary

- **The library is a floor.** Twenty runbooks matched to the twenty failure classes an AI SRE will meet in the first six months of a model-backed product.
- **Shape before content.** Every runbook has the same seven sections — header, detection signals, confirm checklist, mitigation menu, handoff, user communication, fixture back-feed. The shape makes the library legible to a stranger at 3 AM.
- **Mitigate, do not fix.** The AI SRE's runbooks route traffic, open breakers, roll serving configs, and page the peer owner. Root-cause fixes live with the peers.
- **The library is alive.** Review cadence, incident back-feed, and chaos verification keep it honest. A six-month-stale runbook is an incident waiting to happen.

Chapter 02 takes this library of runbooks and sets the command-and-control structure that executes them during a live incident — the IC, the OL, the SMEs, and the comms cadence that AI-flavoured incidents need.
