# The AI-Specific Chaos Experiment Catalogue

## Why this chapter exists

Chapter 01 set the principles. Chapter 02 chose the tooling. This chapter names the experiments — the catalogue of AI-specific chaos the function will actually run. Each entry is a specification: what production fault it reproduces, the hypothesis in steady-state terms, the mechanism (which tool and which primitive), the blast-radius controls, the expected result, and the ties to the runbook library (`mod-108` chapter 01) that this experiment is meant to verify.

The nine experiments below are the floor for an AI chaos program. They are chosen because each exercises a failure class the AI SRE will meet in production, each produces a signal that either validates a runbook or exposes a gap, and each is deliverable with the tooling in chapter 02. Chapter 04 adds the provider-fallback contract experiments as a distinct cluster. Chapter 05 turns the catalogue into a continuous program.

The catalogue is deliberately not twenty. Twenty experiments on a fleet that can run six well is a program nobody executes. The function grows the catalogue only after each existing experiment has run three times, produced a result, back-fed a fixture (`mod-108` chapter 05), and earned its place in the continuous cadence (chapter 05).

The scope here is reliability behaviour. Experiments that inject prompt-injection payloads, jailbreaks, or model-extraction probes are in the `ai-infra-security` peer's catalogue and are explicitly out of scope (chapter 06 of this module).

## 1. The experiment entry shape

Every entry in this catalogue follows the same shape. The shape mirrors the runbook shape from `mod-108` chapter 01 so the two libraries read in the same vocabulary.

- **Name.** A stable slug (`chaos-gpu-eviction-midstream`). Grep-able from the chaos operator's event stream.
- **Production fault model.** The real-world failure the experiment reproduces. One or two sentences, with a pointer to the source (post-mortem, status-page incident, near-miss log).
- **Hypothesis.** Stated in the three steady-state families from chapter 01 §2: serving health, quality, cost. The success criterion is that all three stay within bounds.
- **Mechanism.** The chaos tool and primitive from chapter 02. If the mechanism is a composition, the sequence is listed.
- **Blast-radius controls.** Cohort cap, traffic fraction cap, time cap, auto-halt condition. These are enforced by the mechanism, not by convention.
- **Runbook ties.** Which `mod-108` chapter 01 runbooks this experiment is meant to verify. An experiment with no runbook tie is a candidate for a new runbook — flag it before running.
- **Expected observations.** What the on-call and the dashboard should see while the experiment is live. If the signal is not visible, the observability stack (`mod-104`) has a gap.
- **Fixture back-feed hooks.** Which fixture classes (alert, canary case, eval entry, load-test shape, chaos amendment, runbook update) this experiment's results are expected to feed.

## 2. `chaos-gpu-eviction-midstream` — GPU eviction mid-stream

- **Production fault model.** A higher-priority training job starts in the same cluster and the scheduler preempts a serving pod that is in the middle of streaming a response. In production this is seen as a spike in the preemption counter and a corresponding spike in early-stream-close events (`rb-preemption-spike`, `rb-streaming-stall`).
- **Hypothesis.**
  - **Serving:** streaming-stall rate increases for ≤ 60 seconds; request-success-rate stays within one-hour SLO budget.
  - **Quality:** judge-pipeline scores on completed responses stay within baseline; retried responses do not regress.
  - **Cost:** fallback replica absorbs the traffic within budget; per-request cost stays within 1.3× baseline.
- **Mechanism.** Chaos Mesh `PodChaos` with `action: pod-kill` targeting label `app=serving, chaos.target=true` on a 1% replica slice, during the lowest-traffic window. Alternatively, FIS `aws:eks:pod-delete`. A follow-on variant uses `StressChaos` (memory-hog) on the serving container to reproduce a graceful-preemption trigger under KV pressure rather than an external kill.
- **Blast-radius controls.** 1% of serving replicas, single zone, 10-minute window, auto-halt on 5-minute SLO burn-rate alert or on streaming-stall rate exceeding 3× baseline.
- **Runbook ties.** `rb-preemption-spike`, `rb-streaming-stall`, `rb-ttft-regression` (if the mitigation fails).
- **Expected observations.** Pod-restart event in the k8s event stream; brief stream-close counter spike; router does not route-shift (replica-level eviction, not route-level); PDB respected; scheduler re-places within 60 s.
- **Fixture back-feed.** Alert threshold on streaming-stall rate; chaos-schedule entry; runbook amendment if the actual recovery time exceeded the hypothesis.

## 3. `chaos-provider-timeout-injection` — Model-provider timeout injection

- **Production fault model.** The primary provider stalls mid-request (TCP open, body blocked) for a sustained window. Observed on every major provider; `mod-107` chapter 03's public-incident catalogue has multiple exemplars.
- **Hypothesis.**
  - **Serving:** router breaker opens within the circuit-breaker window (`mod-107` chapter 02 §2); fallback-route traffic ratio climbs to policy level within one minute; request-success-rate dips briefly, recovers under SLO.
  - **Quality:** fallback route's equivalence table holds (`mod-107` chapter 02 §3); judge-pipeline score on the fallback stays within tier-A tolerance.
  - **Cost:** per-request cost on fallback is within the row's `cost_ratio_vs_primary`.
- **Mechanism.** Chaos Mesh `HTTPChaos` with `delay: 30s` targeting egress to the primary provider's API (chapter 02 §4.1). Also run as `abort: true` in a second experiment to cover the 5xx shape. Both shapes needed — the breaker must fire on stalls *and* on errors.
- **Blast-radius controls.** 1% of inference-route traffic, internal cohort first, 15-minute window, auto-halt on fallback-hit-rate below 95% during the window.
- **Runbook ties.** `rb-provider-degraded-fallback`.
- **Expected observations.** Breaker-open event; fallback-route traffic ratio rises; status-page ingestor correlation (status-page may or may not show the primary — the system should not depend on it); fallback-replica latency and judge-score panels on dashboard.
- **Fixture back-feed.** Fallback-contract compliance test (chapter 04); breaker-dashboard panel; fallback-route canary load shape.

## 4. `chaos-prompt-cache-invalidation` — Prompt-cache invalidation / poisoning

- **Production fault model.** Prompt-template change (an application deploy) or tokenizer / model-version change invalidates the stable prefix the serving-layer cache depends on, dropping cache hit rate from a high baseline to near-zero over minutes. Observed as `rb-cache-hit-cliff`; a secondary form is a prefix-cache row being populated with a response from a different tenant (poisoning) if cache-key scoping is wrong, which is a safety-adjacent concern that must be tier-2 (staging-only).
- **Hypothesis.**
  - **Invalidation variant (tier-1).** When the cache is flushed on a route, TTFT regresses by the expected delta (cache-miss fraction × cache-miss latency) and recovers to baseline as the cache warms over the configured warm-up window.
  - **Poisoning variant (tier-2).** When a synthetic cross-tenant key is injected, the cache lookup rejects it (keys are tenant-scoped) and the metric `prompt_cache_cross_tenant_lookup_rejected_total` increments. If the lookup succeeds and returns a wrong-tenant response, the experiment has found a real safety bug.
- **Mechanism.**
  - **Invalidation.** A chaos-wrapper job calls the cache admin API or triggers a cache-key purge on the labelled slice. Chaos Mesh `HTTPChaos` can intercept the cache-service responses to simulate a cold cache.
  - **Poisoning (staging only).** A test harness injects a crafted key via the cache admin API and queries with a different tenant's credentials. The system should fail closed.
- **Blast-radius controls.**
  - **Invalidation.** 5% of a single route, 20-minute window, auto-halt on TTFT exceeding 3× baseline for 5 minutes.
  - **Poisoning.** Staging only. No production cohort.
- **Runbook ties.** `rb-cache-hit-cliff`, `rb-ttft-regression`.
- **Expected observations.** Cache-hit-rate panel drops, TTFT panel rises, cache memory budget shows warm-up, fallback for the slow slice may trigger if latency crosses the breaker threshold.
- **Fixture back-feed.** Template-change CI gate that measures prefix-stability; alert on cache-hit-rate cliff; cross-tenant key-rejection counter on the dashboard.

## 5. `chaos-degraded-mode-routing-drill` — Degraded-mode routing

- **Production fault model.** The primary route is up, but quality (judge-pipeline score, refusal rate, structured-output conformance) has regressed — the "provider degraded but 200" case. The router must demote the route to a lower-tier equivalent and inform the application that capability is temporarily reduced. Observed as `rb-hallucination-rate-spike` or `rb-refusal-rate-spike` on a route without a corresponding 5xx signal.
- **Hypothesis.**
  - **Serving:** a synthetic quality-regression signal injected into the router's health signal causes a demotion to the degraded-mode row of the equivalence table within the policy window (`mod-107` chapter 02 §4).
  - **Quality:** downstream consumers see the capability-reduced flag; the application degrades gracefully (fewer auto-publish paths, higher refusal thresholds); judge-pipeline score on the degraded route is above the tier-B floor.
  - **Cost:** cost-per-request drops (degraded route is cheaper or capped by cap).
- **Mechanism.** Chaos-wrapper injects a synthetic signal into the router's health-signal source (an admin API that marks a route's quality score as "degraded"). HTTPChaos can intercept the judge-pipeline output to the router's breaker if a less invasive injection is wanted.
- **Blast-radius controls.** 1% of traffic for a single route, 15-minute window, auto-halt on any safety-signal spike (not only SLO burn).
- **Runbook ties.** `rb-hallucination-rate-spike` (mitigation menu item 1), `rb-provider-degraded-fallback` (quality-regression branch).
- **Expected observations.** Router demotion event; application-layer "capability reduced" flag; status-page template fires per policy (if the route is customer-facing at the time); safety classifier stays flat.
- **Fixture back-feed.** Runbook amendment on demotion step; alert on router-demotion events; canary test that proves the degraded route handles the previously-passing eval items.

## 6. `chaos-agent-tool-call-fault` — Agent tool-call fault injection

- **Production fault model.** An agent's tool (search API, database call, external service) starts returning a shape the agent retries into — a 429 with an ambiguous body, a 200 with an empty payload, a 500 with a retry-after header shorter than the agent's back-off. The agent cascades (`rb-tool-call-cascade`). Observed in every major agent production deployment in some form.
- **Hypothesis.**
  - **Serving:** per-session tool-call count stays under the configured cap; per-session cost stays under the configured cap; the agent completes or exits gracefully.
  - **Quality:** final response includes a correct "I couldn't retrieve X" signal or an appropriate refusal; judge-pipeline on degraded responses stays within tier.
  - **Cost:** per-session cost does not exceed the chaos-budget alert.
- **Mechanism.** Chaos Mesh `HTTPChaos` on egress to the tool API, configured to return the exact fault shape (429 with empty body, 200 with empty payload, 500 with aggressive retry-after). A follow-on variant uses the agent orchestrator's fault-injection hook if one exists (some agent frameworks have a first-class test-mode shim).
- **Blast-radius controls.** Internal cohort only, 1% of agent sessions, 10-minute window, auto-halt on any per-session cost exceeding 2× baseline or on any side-effecting tool firing (payment, publish, send). The side-effect auto-halt is a chapter 01 §6.1 "no-chaos" protection.
- **Runbook ties.** `rb-tool-call-cascade`, `rb-cost-blow-up`.
- **Expected observations.** Tool-call depth histogram p99 moves; per-session cost panel moves; agent orchestrator's retry log shows the back-off; eventual session termination.
- **Fixture back-feed.** Recursive-tool-call detector regression test; per-session cost cap alert; agent-orchestrator max-depth gate test; runbook amendment with the specific fault shape that caused the loop.

## 7. `chaos-worker-crash-under-stream` — Worker crash under in-flight stream

- **Production fault model.** A serving worker (the process doing model execution on a replica) crashes while it has an open streaming connection. The gateway must detect the drop, close the client stream with a terminal event (not abort), and the application must handle the half-complete response. Observed as `rb-streaming-stall` with the "stream closed without terminal event" signal.
- **Hypothesis.**
  - **Serving:** the client sees a clean terminal event (not an abrupt close); the gateway emits a `stream_terminated_by_server` reason code; the retry path (if the application retries) succeeds on a different replica.
  - **Quality:** post-retry judge-pipeline score is within baseline.
  - **Cost:** the retried request's cost does not double-bill (idempotency semantics from `mod-107` chapter 02 §5 hold).
- **Mechanism.** Chaos Mesh `PodChaos` with `action: container-kill` targeting the worker container (not the sidecar). Timing is critical: the kill must land while a stream is active. A test-harness generates long-running stream requests during the experiment window.
- **Blast-radius controls.** Single pod, single zone, 5-minute window, auto-halt on stream-terminated-without-reason counter rising.
- **Runbook ties.** `rb-streaming-stall`, `rb-preemption-spike`.
- **Expected observations.** Worker-crash event; stream-terminated events with a reason code; retry success on a different replica; no double-billing (if the system has per-request cost attribution).
- **Fixture back-feed.** Alert on stream-terminated-without-reason; canary load generating long streams; retry-idempotency unit test.

## 8. `chaos-kv-pressure-graceful-preemption` — Graceful preemption under KV pressure

- **Production fault model.** A sustained long-context workload fills the KV cache; the scheduler needs to shed memory. The serving layer must preempt sessions gracefully — finish the current step, emit a terminal event, free the KV, allow the scheduler to proceed — rather than kill mid-decode. Observed when prompt-length distribution shifts (`rb-tpot-regression`, `rb-streaming-stall`).
- **Hypothesis.**
  - **Serving:** under memory pressure the scheduler triggers graceful preemption; TPOT regresses on the preempted slice but inter-token gap stays within the streaming-stall SLO on surviving sessions; KV eviction rate rises then returns to baseline.
  - **Quality:** judge-pipeline score on retried requests is within baseline.
  - **Cost:** preempted sessions are not double-billed; cost-per-session metric is accurate.
- **Mechanism.** `StressChaos` (memory-hog) targeting the serving container to synthesise memory pressure; a load generator pushes long-context requests. Alternatively, a scheduler-admin API hook forces graceful-preemption on a labelled replica slice.
- **Blast-radius controls.** 1% of serving replicas, 15-minute window, auto-halt on TPOT exceeding 5× baseline or on TPOT of surviving sessions exceeding 2× baseline.
- **Runbook ties.** `rb-preemption-spike`, `rb-tpot-regression`, `rb-streaming-stall`.
- **Expected observations.** KV-eviction counter rises; preempted-session counter increments; TPOT panel shifts; session-level terminal-event rate stays normal.
- **Fixture back-feed.** Scheduler policy document amendment; alert on KV-pressure threshold; canary long-context load.

## 9. `chaos-judge-model-unavailable` — Judge-model unavailability

- **Production fault model.** The judge-pipeline model (used for online quality signals, canary evals, or production gating) becomes unavailable or starts timing out. If downstream systems read the judge's output uncritically, a judge outage can look like a quality regression (false alarm) or can silently drop production gating. Observed as a secondary effect in multiple AI incidents where online quality was mis-interpreted.
- **Hypothesis.**
  - **Serving:** judge-pipeline health signal declares unavailable within the breaker window; judge-derived alerts are suspended (`rb-judge-model-drift` §mitigation); production gates that depend on the judge enter a documented fail-open or fail-closed state per policy, not a silent one.
  - **Quality:** no false-positive hallucination / refusal alerts during the window; canary runs that need the judge are explicitly paused and surfaced in the dashboard.
  - **Cost:** judge-cost drops to zero on the affected routes.
- **Mechanism.** Chaos Mesh `HTTPChaos` on egress to the judge-model endpoint (abort or long delay). Both shapes needed.
- **Blast-radius controls.** 100% of judge calls on the test slice (the experiment's point is total unavailability); 10-minute window; auto-halt on production-gate failure-mode being wrong (fail-closed when policy is fail-open, or vice versa).
- **Runbook ties.** `rb-judge-model-drift` (the "unavailable" case is a cousin of drift; both suspend judge-derived alerts).
- **Expected observations.** Judge-availability panel drops; dependent alerts show "judge-unavailable" annotation; production gates audit a "judge-unavailable, policy applied: X" line.
- **Fixture back-feed.** Runbook amendment with judge-unavailable mitigation; alert on judge-availability; policy document for fail-open vs. fail-closed per gate.

## 10. `chaos-retrieval-index-drop` — Retrieval-index drop

- **Production fault model.** A vector-index replica fails, an ingest pipeline stalls, or an embedding service goes down, dropping recall@k on the affected slice. Observed as `rb-retrieval-index-staleness` or a harder-cut variant: the index itself is unreachable. The application's RAG route must degrade — either serve a "cannot-retrieve" response, route to a replica, or suppress retrieval entirely.
- **Hypothesis.**
  - **Serving:** retrieval timeouts fire on the affected route; the application's "retrieval unavailable" path is exercised; request-success-rate stays within budget (even if quality is degraded).
  - **Quality:** final response flags the retrieval gap honestly (not a confident wrong answer); judge-pipeline groundedness metric stays within tolerance for the explicit-gap response shape.
  - **Cost:** fallback retrieval source (if any) is within budget.
- **Mechanism.** Chaos Mesh `PodChaos` on the vector-index pod(s), or `NetworkChaos` partitioning the retrieval service from the application. The partition variant covers "retrieval service exists but unreachable"; the pod-kill variant covers "retrieval service is gone." Run both.
- **Blast-radius controls.** Single replica or single pod, 10-minute window, auto-halt on groundedness score dropping below floor (false-confidence is the dangerous mode).
- **Runbook ties.** `rb-retrieval-index-staleness`, `rb-embedding-drift` (adjacent).
- **Expected observations.** Retrieval-service-error panel rises; application's retrieval-gap handling panel shows the explicit-gap responses; judge-groundedness holds (because responses say "I don't have that data").
- **Fixture back-feed.** Retrieval-service-error alert; canary query set that exercises the gap path; application-layer test that the "cannot retrieve" response shape is honest.

## 11. The no-chaos list

Not every surface is a chaos target. The following are **excluded** from experimentation. Every program names its own additions.

- **Billing and accounting ledgers.** A chaos run that makes an audit trail wrong is a legal event.
- **Model registry write paths.** Weight corruption is already a SEV-1 failure class (`rb-weight-corruption`); the chaos program verifies the detection and rollback, not the corruption itself.
- **Moderation-queue integrity.** The safety / policy peer owns this; chaos through the queue would create a window where safety signals are untrusted.
- **Live customer agent sessions with side-effecting tools unmuted.** Side effects (`send_email`, `pay_invoice`, `publish_post`) must be stubbed during any agent-adjacent chaos.
- **Payment, publish, send endpoints reached through any tool, even in staging-adjacent cohorts.** The blast radius exceeds what a chaos policy can contain.
- **Any surface the security peer owns.** Chaos against security controls is `ai-infra-security`'s program, with its own policy.

The no-chaos list is appended to the chaos-program policy (chapter 05 §5) and is reviewed quarterly. Removing an entry requires the safety / policy peer's signature.

## 12. The catalogue-authoring worksheet

The exercise set in exercises 03, 04, and 05 of this module authors environment-specific versions of these experiments. The worksheet shape:

1. **Name the fault.** One sentence. Trace it to a post-mortem or near-miss.
2. **Write the three-family hypothesis.** Serving, quality, cost. All three.
3. **Pick the mechanism.** From chapter 02 §4's tool-to-catalogue map. Note deviations.
4. **Size the blast radius.** Cohort, fraction, time, auto-halt. Mechanical enforcement.
5. **Tie to the runbook.** Name the `mod-108` chapter 01 runbooks this verifies. If none exists, open a runbook-authoring task first.
6. **Name the expected observation.** If the dashboard does not show it, surface the `mod-104` gap.
7. **Commit the fixture back-feed hooks.** Which fixture classes the result flows into.

## Summary

- The AI chaos catalogue has a floor of nine experiments: GPU eviction mid-stream, provider-timeout injection, prompt-cache invalidation / poisoning, degraded-mode routing drill, agent tool-call fault, worker crash under in-flight stream, KV-pressure graceful preemption, judge-model unavailability, retrieval-index drop.
- Every entry has the same shape: name, production fault model, three-family hypothesis, mechanism, blast-radius, runbook ties, expected observations, fixture back-feed.
- The no-chaos list is as important as the catalogue. Billing, registry writes, moderation queues, side-effecting tools on live sessions, and any security-peer surface are out.
- Each experiment's results flow back to the fixture ledger (`mod-108` chapter 05) and the runbook library (`mod-108` chapter 01). A chaos result that produces neither is a chaos result that was not interpreted.

Chapter 04 takes the provider-timeout experiment as the seed and expands it into the full provider-fallback contract chaos program.
