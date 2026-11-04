# Cross-Peer Interface Contracts: Performance, Inference-Engineer, Security, ML-Platform, Eval, MLOps

## Why this chapter exists

Every chapter in this track has, in its final section, named the peer boundaries that bound the chapter's scope. Mod-107 chapter 07, mod-108 chapter 07, mod-109 chapter 06, mod-110 chapter 06, mod-111 chapter 08, mod-112 chapter 07, mod-113 chapter 07 — each writes "here is who owns what, here is what the function hands up, here is what the peer hands back." The pattern is consistent and the vocabulary is shared.

At team scope, those per-module boundaries need to be composed into a single operating contract. A reliability function that re-reads seven peer-boundary chapters every time a cross-track question arrives is a function that is doing routing work the organisation expected the function to systematise. The **interface contract** is that systematisation — a Git-tracked, signed document, one per peer track, that writes the shared vocabulary, mutual deliverables, SLA for each deliverable, and the escalation path when the contract is in dispute.

This chapter writes the interface-contract shape and then walks the six contracts the AI SRE function owns at team scope: with `ai-infra-performance`, `ai-infra-inference-engineer`, `ai-infra-security`, `ai-infra-ml-platform`, `ai-eval-engineer`, and `ai-infra-mlops`. The chapter does **not** re-litigate the per-module boundaries — those remain the authoritative detail; this chapter writes the *interface layer* over them.

Primary references: mod-101 chapter 02 (the peer deferral map) for the vocabulary; the per-module peer-boundary chapters already cited for the detail; mod-107 chapter 02 for the fallback-contract shape this chapter's interface contracts generalise; the OpenTelemetry [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) and [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/) telemetry surface as the signal substrate across most of the interfaces.

## 1. What an interface contract is

An interface contract is a one-page (occasionally two-page) Git-tracked document between the AI SRE function and one peer track. It specifies:

- **Vocabulary.** The shared terms that mean the same thing on both sides of the interface. "Latency," "regression," "incident," "degraded," "release-gate" — each has an operational definition both sides consume.
- **What the SRE owes.** The deliverables the function produces for the peer: signal surfaces, tickets, dashboards, auto-halt wiring, drill invitations, incident-command handoffs.
- **What the peer owes.** The deliverables the peer produces for the function: artifacts, telemetry, release-gate signals, joint post-mortems, review attendance.
- **SLAs on each deliverable.** The cadence or the response-time commitment attached to each. "The eval team's canary-eval verdict will land within 90 minutes of a release-gate query" is an SLA; "the eval team will respond quickly" is not.
- **Dispute escalation.** The path when the contract is in dispute — ticket backlog grows, SLA is missed, the deliverable's quality drifts. First the two leads talk; then the architect; then the director.
- **Review cadence and signoffs.** The contract is reviewed quarterly (minimum) or whenever the peer's or the function's charter changes materially. Both leads sign in Git.

The contract is not an SLA document in the vendor sense — nobody is paying anyone — but it functions as one. A dispute about whether a signal was supposed to be delivered is resolved by reading the contract, not by debate.

## 2. The AI SRE ↔ `ai-infra-performance` contract

### 2.1 Shared surfaces

Kernel regressions, GPU-telemetry correlation, long-tail latency root-causing. The performance peer's craft is kernel-authoring (CUDA, CUTLASS, CuTe DSL, Triton, Nsight-Compute to SASS); the function's surface is the fleet-wide latency SLO that the kernels compose into.

The seam: the performance peer owns how fast one operation runs on one GPU under one workload shape; the function owns whether the fleet meets its SLO under its real traffic mix. A regression inside a single kernel is the peer's; a regression that only surfaces under production batch shapes is a joint ticket.

### 2.2 What the SRE hands to the peer

- **Kernel-regression report.** When the function's dashboards show a performance regression after a known kernel / engine update, the SRE files a *reproducible* ticket:
  - the kernel or engine version delta (before / after SHAs);
  - the workload slice where the regression shows (sequence length, prompt distribution, batch shape, concurrency);
  - the GPU-telemetry surface at the time of regression (DCGM SM activity, memory bandwidth, HBM pressure, per-kernel occupancy — [DCGM metrics reference](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html) by name);
  - the latency surface (TTFT, inter-token latency, P50 / P95 / P99, broken out per model and per route);
  - a minimum reproducible test case where possible, or a pointer to a saved trace pack if not.
- **Nsight-level profiling ask.** When the function needs profiling depth it does not have — a specific operator regression, a tensor-core utilisation question, a memory-access-pattern question — the SRE files a *profiling ask* with the above context plus:
  - the hypothesis the profiling is meant to confirm or deny;
  - the service-level impact if the hypothesis is correct;
  - the acceptable turnaround window (not every profiling ask is an incident).
- **Fleet-mix context.** Periodically (quarterly), the function hands the peer an updated fleet-mix snapshot — model-version distribution, traffic-shape distribution, concurrency distribution, batch-size distribution — so the peer's kernel tuning targets the real workload, not a synthetic one.

### 2.3 What the peer hands to the SRE

- **Kernel release notes.** Each kernel or engine update that reaches production is accompanied by release notes naming the intended performance delta (direction and magnitude) and the workload slice where it is expected to show. The function gates the release on whether the production dashboard replicates the expected signal; the peer gets the gate signal back.
- **Regression triage.** When an SRE-filed regression report comes in, the peer acknowledges within 24 hours (business hours), owns the root-cause investigation, and either produces a fix, a rollback, or a "this is expected" response with justification.
- **Pre-production benchmark surface.** The peer runs a canonical benchmark suite (Nsight profiles across a reference workload) on each engine / kernel candidate; the function can read the benchmark surface before production rollout to anticipate SLO impact.

### 2.4 SLAs

- Kernel-regression ticket acknowledgement: 24 business hours.
- Kernel-regression triage decision (owned / declined / escalated): 5 business days.
- Nsight-profiling-ask turnaround: workload-dependent; the SRE and peer negotiate at ticket filing, with a default of 10 business days for non-urgent asks.
- Pre-production benchmark surface update: before each kernel / engine release.
- Fleet-mix context refresh from the SRE: quarterly.

### 2.5 Escalation path

Peer lead ↔ SRE lead first; `ai-infra-architect` second (composition of fleet-wide latency SLO with per-service kernel budget); `ai-infra-senior-architect` third (multi-year performance roadmap composing with multi-year SRE roadmap).

### 2.6 Anti-patterns

- **The SRE files a non-reproducible performance ticket** ("latency feels slower this week"). The peer cannot act; the function wastes the peer's time; the relationship erodes.
- **The peer ships a kernel update without release notes.** The SRE sees a latency delta in production with no advance notice; the deployment-change calendar did not carry the entry; the fix is retroactive.
- **The function silently tunes HPA or fleet-mix in a way that invalidates the peer's benchmark.** The peer tuned for an old workload mix; the function is now operating on a different mix; the regression is real and neither side can explain it.

## 3. The AI SRE ↔ `ai-infra-inference-engineer` contract

### 3.1 Shared surfaces

The serving engine (vLLM / SGLang / TensorRT-LLM / Dynamo / Triton) is the peer's craft; the serving envelope — knobs chosen, capacity sized, routing wired, release gated — is the function's. Mod-111 chapter 08 §1.1 wrote the per-chapter boundary; this contract is the ongoing one.

### 3.2 What the SRE hands to the peer

- **Production-signal bug reports** with the shape in mod-101 chapter 02 (workload, knob matrix, broken SLO, repro conditions).
- **Engine-knob proposals.** When the function proposes changes to engine knobs (max batch, prefix-cache sizing, speculative-decoding config, tensor-parallel degree), the proposal is in writing, references the signal that motivates it, and the peer reviews before production rollout.
- **Engine-update acceptance signal.** Mod-111 chapter 05's release-gate consumes the engine's `X-Model-Version` header and the engine's weight-load-time telemetry; the SRE confirms that the fleet rolls out the update successfully and emits the gate signal back.
- **Chaos / drill invitations.** When a chaos experiment exercises engine primitives (drain, warm, in-flight swap), the peer is on the invitation list; the deployment-change calendar flags the window.

### 3.3 What the peer hands to the SRE

- **Operational primitives.** Drain, warm, in-flight swap, graceful shutdown, health probe semantics — each documented with the invariants the function's orchestration can rely on. Mod-111 chapter 04 §5.2's orchestration contract depends on this.
- **Observability surface.** The engine's own metrics (per-request latency, batch depth, KV-cache utilisation, prefix-cache hit rate) emitted on a Prometheus endpoint and following OpenTelemetry [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) where applicable.
- **Weight-load-time telemetry.** The signal mod-113 chapter 02 §4's cold-start RTO depends on and mod-111 chapter 04 §3's capacity planning for green depends on. The peer owns the measurement; the SRE owns the capacity decision.
- **Pre-release notes.** Each engine release includes notes on: the SRE-visible contract changes (new knobs, new headers, new metrics), the expected operational delta (improved drain time, worse memory footprint, etc.), the known regressions (if any), the required rollback path.
- **Engine-side on-call.** For regression windows (two weeks post-release), the peer stands up an informal on-call so the SRE can escalate engine-internal questions quickly.

### 3.4 SLAs

- Production-signal bug acknowledgement: 24 business hours.
- Operational-primitive contract change: 2-week pre-notification to the SRE.
- New engine-release pre-notes: 1-week pre-notification to the SRE for minor releases, 2-week for major.
- Engine-side on-call window post-release: 2 weeks (business hours).

### 3.5 Escalation path

Peer lead ↔ SRE lead first; `ai-infra-architect` for engine-portfolio composition (if the organisation runs more than one engine family); senior architect for engine-portfolio strategy.

### 3.6 Anti-patterns

- **The SRE rewrites the engine's drain code** because the primitive is missing. Mod-111 chapter 08 §3.1 named this; the function escalates and uses a slower pattern in the interim, does not shadow-implement.
- **The engine team ships a knob change without updating the shared contract.** The SRE's autoscaler assumes the old semantics; a regression lands; the post-mortem exposes the silent contract drift.
- **The function debates engine internals with the peer.** The internals are not the function's to debate; the operational envelope is. Scoping the ticket to the envelope keeps the conversation productive.

## 4. The AI SRE ↔ `ai-infra-security` contract

### 4.1 Shared surfaces

Prompt-injection classifiers, PII-filter sidecars, jailbreak classifiers, model-extraction query-rate control, ML-BOM signing (mod-110 chapter 03). Each security control sits in the request path and has a latency / cache / throughput signature the function's SLOs read.

### 4.2 What the SRE hands to the peer

- **Latency-impact reports.** When a security control's latency cost moves the function's SLO, the SRE files a report with the latency decomposition (per-stage traces), the magnitude of impact, the service-tier of the affected route, and the error-budget burn implication.
- **Fail-open vs. fail-closed decisions in writing.** For each security control, the function documents the production fail-mode in the service's runbook (mod-108 chapter 01). The security peer countersigns.
- **Joint post-mortem invitations.** Any incident where a security control was in the causal path — control failed open, control failed closed and cascaded into availability loss, control degraded and consumed latency budget — is a joint post-mortem (mod-110 chapter 06). The function initiates; the peer attends.
- **Chaos / drill invitations.** Mod-109's chaos program includes experiments against security controls (control-sidecar failure, classifier latency spike). The security peer is on the invitation list.

### 4.3 What the peer hands to the SRE

- **Documented latency budget per control.** Each control ships with a stated P50 / P95 / P99 latency envelope under reference load. The SRE's capacity planning (mod-106) consumes this.
- **Cache-impact profile.** Where a security control changes the request body (PII redaction), the control's cache-key impact is documented; the function's prefix-cache (mod-104) planning depends on it.
- **Fail-open vs. fail-closed policy.** Per control, the peer states the intended fail mode, with the reliability / safety trade-off written explicitly.
- **Co-ownership on joint post-mortems.** The security peer attends and contributes the security-side forensics; the function owns IC.
- **Threat-model updates.** When the security posture changes (new control, control retirement, policy change), the SRE is consulted before rollout; the deployment-change calendar carries the entry.

### 4.4 Shared latency budget

For high-SLO routes, the function and the security peer agree on a joint *shared* latency budget — N milliseconds of end-to-end latency, allocated between inference, security, retrieval, and application layers. The function is responsible for the inference and retrieval slices; the peer is responsible for the security slice. Each side manages its slice; neither side unilaterally eats the other's.

The shared budget is a chapter 05 upward-communication tool: when the product lead asks why latency went up, the function points to the slice responsible, which has an owner.

### 4.5 SLAs

- Latency-impact report acknowledgement: 1 business day for tier-1 routes, 3 for tier-2.
- Joint post-mortem attendance: within 72 hours of incident close.
- Pre-rollout consult for new or changed controls: 1 week minimum.
- Threat-model update review: quarterly, or on charter change.

### 4.6 Escalation path

Peer lead ↔ SRE lead first; architect second (cross-service security-reliability composition); legal / compliance third (regulatory or contractual implications per mod-110 chapter 06).

### 4.7 Anti-patterns

- **The function silently bypasses a security control** to meet an SLO. The post-mortem will surface the bypass; the trust the two functions have built is damaged.
- **The peer deploys a new control without the deployment-change-calendar entry.** The function sees a latency step-change with no advance context; triage is slower than it needs to be.
- **The function treats a joint incident as a security-only or reliability-only post-mortem.** The two sides have different lessons; neither alone produces the correct action items.

## 5. The AI SRE ↔ `ai-infra-ml-platform` contract

### 5.1 Shared surfaces

The ML-platform peer (level 30) owns the self-service ML platform, model registry, experiment-tracking, training-cluster plumbing, and the promotion-to-serving workflow. The function owns the production envelope the promotion lands into. Mod-111 chapter 08 §1.2 wrote the chapter-local boundary; this contract is the running one.

### 5.2 What the SRE hands to the peer

- **Reliability acceptance criteria for promotion.** Per service, the function states the SLO the candidate must meet in shadow or canary before promotion to primary (mod-111 chapters 02, 03).
- **Promotion-manifest requirements.** The manifest fields the function reads to operate the promotion safely: artifact URI, artifact hash, model-version string, expected memory footprint, expected throughput at reference concurrency, rollback-target version.
- **Rollback-path confirmation.** Each promotion includes a documented rollback path the function has verified in a pre-production drill.
- **Production-signal feedback.** When a promotion lands and the fleet behaves unexpectedly, the SRE files back with the signal: the SLO that moved, the fleet-mix at the time, the suspected root-cause slice (artifact-side or envelope-side).

### 5.3 What the peer hands to the SRE

- **Artifact lineage.** Each artifact has traceable lineage: the training-run ID, the dataset snapshot, the fine-tune parameters, the eval verdict, the signer identity. Mod-110 chapter 03's ML-BOM composes with this.
- **Eval evidence.** The eval verdict from the `ai-eval-engineer` peer (§7 of this chapter) is attached to the artifact; the SRE reads it at release-gate evaluation.
- **Rollback hook.** A documented mechanism the function invokes to roll back a promotion — a registered rollback target, an API call, an atomic-pointer swap. The hook is tested on each major platform-pipeline release.
- **Promotion-pipeline SLA.** The time from artifact ready to production-available, with the pipeline's own error-budget. If the pipeline is itself unreliable, the function needs to know.

### 5.4 SLAs

- Reliability-acceptance-criteria review at platform-pipeline changes: 2 weeks pre-rollout.
- Artifact-lineage completeness per promotion: 100%; a promotion with incomplete lineage is blocked at the release gate.
- Rollback-hook validation: per major release; the function participates.
- Production-signal-feedback acknowledgement: 1 business day.

### 5.5 Escalation path

Peer lead ↔ SRE lead first; architect second (fleet-level artifact posture, model-version-pin policy composition with mod-107 chapter 06); senior architect third (multi-year ML-platform roadmap composition).

### 5.6 Anti-patterns

- **The function tries to own the model-registry's schema.** It is the peer's. The function's influence is through the reliability-acceptance-criteria contract.
- **The platform ships a new promotion pipeline without a working rollback hook.** The first production-side rollback becomes a crisis; the trust breaks.
- **The function accepts a promotion without the eval-evidence attached.** The release-gate is bypassed; the subsequent regression is a joint incident but was preventable.

## 6. The AI SRE ↔ `ai-eval-engineer` contract

### 6.1 Shared surfaces

The eval peer (level 30) owns the methodology for evaluating model quality — offline eval set authoring, canary-eval set curation, judge-pipeline design, benchmark selection, calibration against human labels, the regression-set ledger, quality threshold-setting. Mod-111 chapter 08 §1.3 wrote the chapter-local boundary; this contract is the ongoing one.

### 6.2 What the SRE hands to the peer

- **Release-gate contract.** Mod-111 chapter 05's release-gate consumes the eval verdict against a signed schema. The SRE owns the release-gate service; the eval peer owns the verdict content.
- **Judge service reliability.** The eval peer's judge-pipeline runs as a service the SRE operates — SLO, latency, capacity, failover. The SRE treats the judge as a dependency the same way it treats any other dependency.
- **Drift-alert subscription.** The SRE subscribes to the eval peer's drift-alert signal; drift alerts feed the monthly SLO review's quality section (chapter 04).
- **Joint post-mortem invitations** for incidents where quality was a causal factor.

### 6.3 What the peer hands to the SRE

- **Release-gate signal.** The eval verdict per candidate, against the signed release-gate schema, with per-check pass / fail and per-check evidence links.
- **Drift alert.** The ongoing signal that the production distribution has shifted away from the eval distribution. The alert fires through the function's alerting pipeline; the function's on-call triages and routes.
- **Judge calibration metadata.** The judge's calibration-against-humans trend, the confidence intervals, the known failure modes. The SRE reads this to interpret the release-gate verdict and the drift alerts correctly.
- **Threshold-change review.** Changes to eval thresholds that affect the release-gate verdict are reviewed with the SRE before rollout; the deployment-change calendar carries the entry.

### 6.4 SLAs

- Release-gate verdict for a submitted candidate: 90 minutes for tier-1 routes, 4 hours for tier-2. (Specific numbers belong to the team's contract; this chapter illustrates the shape.)
- Drift-alert actionability: per alert, with a documented triage runbook maintained by the eval peer.
- Threshold-change pre-notification: 2 weeks minimum.
- Joint post-mortem attendance: within 72 hours of incident close.

### 6.5 Escalation path

Peer lead ↔ SRE lead first; architect second (release-gate composition across services); product third (when a threshold change has a business-impact dimension beyond reliability).

### 6.6 Anti-patterns

- **The function tunes judge prompts or calibration unilaterally.** It is the peer's domain; mod-111 chapter 08 §2 is explicit.
- **The eval peer treats the judge service as "someone else's problem" to operate.** The judge *is* a production service; it has an SLO; the eval peer contributes to the on-call model for it.
- **The function reads the drift alert without understanding the judge's calibration.** False-positive storms waste the on-call; false negatives let real regressions through.

## 7. The AI SRE ↔ `ai-infra-mlops` contract

### 7.1 Shared surfaces

The mlops peer (level 25) owns the CI/CD pipeline for the service image, model artifact, config, and retrieval index. The function owns the production envelope and the release-gate service the pipeline calls into. Mod-111 chapter 08 §1.2 and mod-112 chapters 01–06 write the detail.

### 7.2 What the SRE hands to the peer

- **Release-gate service API.** A stable endpoint (`GET /verdict/{candidate_sha}`, `POST /verdict`) with a versioned schema the pipeline calls.
- **DORA four-keys instrumentation.** The telemetry mod-111 chapter 06 needs to compute deployment frequency, lead time for changes, change-failure rate, MTTR. The SRE emits; the mlops peer aggregates or mirrors.
- **Pipeline-reliability signal.** When the mlops pipeline itself is slow, broken, or flaky, the SRE surfaces the signal through the monthly SLO review; the mlops peer's pipeline has its own reliability contract.
- **Audit-log retention policy** (mod-110 chapter 05) that the pipeline emits against.

### 7.3 What the peer hands to the SRE

- **Audit log of releases.** Who promoted what, when, with what evidence, through what pipeline — the mod-111 chapter 04 §4.4 record that mod-111 chapter 06's DORA report reads.
- **Pipeline-SLA signal.** The pipeline's own observability — build success rate, build duration, deploy duration, deploy success rate — exposed to the function.
- **Candidate pre-flight signal.** Before the release-gate is called, the pipeline attaches: artifact hash, lineage evidence, pre-flight scan results (vulnerability, policy).
- **Change-admission compliance.** The pipeline respects the deployment-change calendar's windows (chapter 01 §3.2); red-window attempts are blocked at the pipeline, not at the release-gate.

### 7.4 SLAs

- Pipeline integration against release-gate schema changes: 1 week pre-rollout notice.
- DORA-signal completeness: 100% of releases instrumented.
- Audit-log retention: per the mod-110 chapter 05 policy.
- Change-window compliance: 100%; exceptions require named-individual signoff.

### 7.5 Escalation path

Peer lead ↔ SRE lead first; architect second (pipeline composition across teams); senior architect third (multi-year CI/CD strategy).

### 7.6 Anti-patterns

- **The function rewrites the CI/CD pipeline** because the pipeline is slow. Mod-111 chapter 08 §3.2 is explicit; the SRE's leverage is the release-gate interface and the DORA signal.
- **The mlops peer adds a bypass that skips the release-gate.** The audit log will catch it; the trust breaks. The pipeline must call the gate even when the gate's verdict is expected to pass.
- **The function emits DORA metrics in a way the pipeline cannot confirm.** The numbers diverge between function dashboards and pipeline dashboards; leadership stops trusting either.

## 8. The composition artifact

Each of §2–§7 produces a one-page contract. The function keeps the six contracts in a single `interfaces/` directory in the team's Git repository, with an index that lists each peer, the contract's current version, the last review date, and the current open escalations (if any).

The composition artifact — the top-level `INDEX.md` — is the document the function hands to:

- A new team lead on day one (chapter 01 §7).
- The architect peer when roadmap composition is in question (chapter 05 §2).
- An auditor when the audit asks "how does your team interact with the other teams?" (mod-113 chapter 07 §4.1).

## 9. The cross-service SLO composition problem

Several of the interface contracts compose. A single production incident can involve the engine (§3), a security control (§4), an ML-platform artifact (§5), and an eval drift (§6) simultaneously. The function's SLO is the composition — the user sees one latency, one error rate, one quality level, regardless of which peer owns which slice.

Three composition patterns the function uses:

- **Per-slice error budget.** Decompose the SLO into per-slice budgets (inference, security, retrieval, application). Each peer's slice has an owner. Chapter 05 §2 writes the upward-communication version of this.
- **Causal attribution at post-mortem.** Mod-108 chapter 04's post-mortem structure includes a per-slice causal attribution for the incident; the attribution closes the loop back to the responsible peer.
- **Joint drill design.** Mod-109's chaos program includes experiments that deliberately cross interfaces — a security-control latency spike and a cache-invalidation event simultaneously, for instance. The drill exercises the composition; the function and all affected peers attend.

A function that writes the six contracts but does not operate the composition is a function whose peers each individually feel supported and whose SLOs still slip. The composition is where the function's team-scope leadership shows up.

## 10. The quarterly peer-interface health check

At each quarterly retrospective (chapter 01 §2.4), the function runs a health check against each of the six interfaces:

- **SLA attainment per deliverable.** Did the peer hit the agreed SLAs? Did the function?
- **Open ticket count.** How many SRE-filed tickets with the peer are open; how many peer-filed tickets with the SRE.
- **Open escalation count.** Any disputes that escalated to architect or above in the quarter.
- **Shared-incident count.** How many joint post-mortems ran; what classes of incident.
- **Contract review cadence.** Is the contract itself current, or has a charter change invalidated part of it.

The health check is a 5–10-minute item per interface. A declining interface — missed SLAs, growing escalation count, aging tickets — is a signal the function raises at the annual programme review (chapter 01 §2.5) and brings into the senior-architect conversation (chapter 05 §3).

## Summary

- An interface contract is a one-page Git-tracked document between the AI SRE function and one peer track: vocabulary, mutual deliverables, SLAs per deliverable, escalation path, review cadence. The six peer contracts — performance, inference-engineer, security, ML-platform, eval, mlops — compose the function's horizontal operating surface.
- The performance contract centres on kernel-regression reports from the SRE, kernel release notes from the peer, and joint Nsight-profiling asks on ambiguous regressions.
- The inference-engineer contract centres on engine operational primitives (drain, warm, in-flight swap), engine observability surface, weight-load-time telemetry, and engine-side on-call post-release.
- The security contract centres on the shared latency budget, fail-open vs. fail-closed policy per control, joint post-mortems, and pre-rollout consultation on control changes.
- The ML-platform contract centres on artifact lineage, eval evidence attached to promotions, rollback hooks, and the reliability acceptance criteria the platform's pipeline respects.
- The eval contract centres on the release-gate verdict, drift alerts, judge calibration metadata, and threshold-change pre-notification. The judge-as-a-service is operated by the SRE but its content is owned by the peer.
- The mlops contract centres on the release-gate service API, the audit log, DORA signals, pipeline-reliability signal, and change-window compliance.
- Cross-service SLO composition is where the function's team-scope leadership shows up: per-slice error budgets, causal attribution at post-mortem, and joint drill design.
- A quarterly peer-interface health check keeps the six contracts honest; declining interfaces surface into the annual programme review and the senior-architect conversation.

Chapter 04 writes the monthly SLO review ritual — the forum in which many of the peer-interface signals above get read together and acted on.
