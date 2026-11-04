# Peer Boundaries for Release Engineering

## Why this chapter exists

Release engineering on an AI service is a joint practice. Weight updates ship through pipelines that are not owned by the AI SRE, through serving processes whose internals are not owned by the AI SRE, past evaluation gates whose methodology is not owned by the AI SRE. The function's temptation — because the operational symptoms show up on the function's dashboards — is to grow into every adjacent surface: to rewrite the engine's drain primitive, to re-author the CI/CD pipeline, to redesign the eval harness. Each of those is a different discipline with its own expertise, and the function that tries to own all three is the function that owns none of them well.

This chapter writes the boundary. It names three peer disciplines whose work composes with the module, describes what each owns and what the AI SRE *does not* own, and writes the shared artifacts where the peers and the AI SRE must collaborate. It mirrors `mod-107` chapter 07, `mod-108` chapter 07, `mod-109` chapter 06, and `mod-110` chapter 06 in function and tone: vocabulary first, handoffs second, anti-patterns last.

The chapter is the module's closing discipline. Everything the module built — canary, shadow, blue-green, release gate, DORA, chaos+load pairing — depends on these peer relationships being drawn cleanly. A release-engineering program that drifts past the boundary is a program whose quality the function cannot defend.

## 1. The three peers at the release surface

Three functions sit next to the AI SRE on the release path. Each owns a distinct part of the pipeline; each has a specific handoff to the SRE.

### 1.1 `ai-infra-inference-engineer` peer

- **What they own.** The inference engine's runtime behaviour — the KV-cache manager, the batching scheduler, the tensor-parallel group, the streaming event emitter, the engine-side drain primitive, the in-flight request swap, in-place weight update if supported. The engine-side mechanics that the AI SRE's orchestration reads but does not implement.
- **Naming per `mod-101`.** The peer is `ai-infra-inference-engineer` (level 25 in the engineering ladder). <!-- needs-research: confirm the level number against the current ladder document; the explicit number may differ -->
- **Representative tooling.** vLLM, TensorRT-LLM, Text Generation Inference (TGI), NVIDIA Triton, llama.cpp, LMDeploy. Each has its own drain / warm / swap semantics.
- **Handoff from this module.** The orchestration contract (chapter 04 §5.2) depends on a drain primitive the engine exposes. The release-gate contract (chapter 05) depends on the engine's `X-Model-Version` header being emitted. The capacity planning for green (chapter 04 §3) depends on the engine's weight-load-time telemetry.
- **Handoff to this module.** The engine does not write the release pipeline; the engine exposes primitives the release pipeline calls.

### 1.2 `ai-infra-mlops` peer

- **What they own.** The CI/CD pipeline for the service image, model artifact, config, and retrieval index. The Git-to-artifact promotion plumbing, the build cache, the artifact registry, the feature-store ingest pipeline, the model-registry write path. The path that produces the candidate the release-gate contract evaluates.
- **Naming per `mod-101`.** The peer is `ai-infra-mlops` (level 25). <!-- needs-research: confirm level and exact slug -->
- **Representative tooling.** Argo CD, Argo Workflows, Jenkins, GitHub Actions / GitLab CI, Tekton; MLflow, Weights & Biases, Vertex AI pipelines; model-registry (MLflow, SageMaker, Vertex), feature-store (Feast, Tecton), object-store artifact paths.
- **Handoff from this module.** The release-gate contract (chapter 05) is read by the mlops peer's pipeline; the mlops peer calls into the release-gate service to fetch the verdict and gate promotion.
- **Handoff to this module.** The audit log of releases (chapter 04 §4.4) is written by the mlops peer's pipeline; the DORA report (chapter 06) reads it.

### 1.3 `ai-eval-engineer` peer

- **What they own.** The methodology for evaluating model quality — offline eval set authoring, canary eval set curation, judge-pipeline design, benchmark selection, calibration against human labels, the regression-set ledger, the quality threshold-setting discipline. The function that answers "is this candidate good enough to ship?"
- **Naming per `mod-101`.** The peer is `ai-eval-engineer` (level 30). <!-- needs-research: confirm level and exact slug -->
- **Representative tooling.** Internal eval harnesses; public frameworks (OpenAI Evals, Inspect, Promptfoo, LangSmith eval, Weights & Biases evaluations); judge-model stacks (LLM-as-judge with calibration against human labellers).
- **Handoff from this module.** The release-gate contract (chapter 05 §3.1) consumes the eval verdict; the response-diff pipeline (chapter 02 §2.3) depends on the judge that the eval peer owns.
- **Handoff to this module.** The eval team does not operate the release pipeline; they emit a verdict against a schema and sign off on threshold changes.

## 2. What this module does *not* do

A concrete list beats an abstract principle. The AI SRE's release-engineering function explicitly does not:

- **Write the inference engine's drain, warm, or in-flight-swap code.** The engine's internals are the engineer peer's. The module specifies the contract the engine must satisfy; the engine owns the implementation.
- **Rewrite the CI/CD pipeline.** The mlops peer's pipeline is a dependency, not a surface this module owns. Where the pipeline needs to call out to the release-gate service, the SRE provides the endpoint; the pipeline itself is at mlops.
- **Author eval sets or judge prompts.** The eval peer's methodology is not debatable from inside this module. The SRE wires the verdict into the gate; the SRE does not edit the rubric.
- **Score model quality.** Ever. A regression debate at release time that resolves on "is the new model's response really worse?" is a question for the eval peer with the SRE in a supporting role, not a question the SRE answers unilaterally.
- **Decide the cadence of model training or fine-tune runs.** The mlops and model-training peers determine when a new candidate exists; this module responds to candidates, it does not schedule them.
- **Author provider-backed model-version pinning policy.** The `mod-107` chapter 06 program covers provider-release discipline. The SRE's release-gate consumes a pinned candidate; who pinned it and why is at mod-107.
- **Set retention policy on release-gate verdicts or audit logs.** `mod-110` chapter 05 writes the retention discipline; the release-engineering function emits the records against that discipline.

A release-engineering program that touches any of these surfaces unilaterally has drifted past its charter. The anti-patterns in §4 name the specific drift modes.

## 3. The shared surfaces (where the peers meet)

Four surfaces have shared ownership. On each, the AI SRE is a producer, a consumer, or both.

### 3.1 The engine's drain primitive

- **Who owns.** The engineer peer owns the implementation; the SRE owns the orchestration that invokes it.
- **SRE's role.** Specify the drain contract the orchestration needs (chapter 04 §5). Verify, not assume, the primitive works on each engine version.
- **Boundary clarification.** The SRE does not re-implement the primitive when the engine lacks it; the SRE escalates to the engineer peer with a concrete reproducible test case and, in the interim, uses a slower pattern (full re-deploy from blue-green rather than in-place swap).

### 3.2 The release-gate contract

- **Who owns.** This module authors it (chapter 05); the eval peer signs the eval predicate; the mlops peer's pipeline enforces it.
- **SRE's role.** Keep the contract current (chapter 05 §7), coordinate the quarterly review, mediate between eval's and SRE's thresholds.
- **Boundary clarification.** The contract is Git-tracked, signed by all three peers, and reviewed quarterly. A unilateral edit from any one peer is a change-control violation.

### 3.3 The CI/CD-to-gate interface

- **Who owns.** The mlops peer owns the CI/CD pipeline; the SRE owns the release-gate service it calls into.
- **SRE's role.** Provide a stable API for verdict fetch (`GET /verdict/{candidate_sha}`) and verdict write, with a published schema (chapter 05 §3) and a versioned contract.
- **Boundary clarification.** The SRE does not add release-gate decisions into the pipeline directly; the pipeline reads the gate's verdict. The pipeline's internal steps (build, test, scan, promote) are at mlops; the gate's decision is at the SRE.

### 3.4 The judge pipeline

- **Who owns.** The eval peer owns the judge model, prompt, and calibration; the SRE owns the deployment and reliability characteristics of the judge as a service.
- **SRE's role.** Treat the judge as a dependency (chapter 02 §7) — availability SLO, latency SLO, cost budget, failover. Emit signals the gate reads.
- **Boundary clarification.** The SRE does not tune the judge prompt or change the calibration; those go through the eval peer. The SRE does set the judge service's capacity, autoscaler, and failover behaviour.

### 3.5 The DORA report

- **Who owns.** This module computes and publishes it (chapter 06); the mlops peer provides the audit-log data; the eval peer provides the eval-verdict-pass-rate data; the engineer peer's primitives contribute to MTTR branches.
- **SRE's role.** Produce the quarterly narrative; convene the cross-peer review of trends.
- **Boundary clarification.** The DORA numbers are a function-level metric, not an individual-peer metric. "Model-version change-failure rate is 25%" is a joint signal, not a fault attributed to one peer.

## 4. Anti-patterns at the release boundary

The failure modes the AI SRE must recognise and prevent:

- **The SRE rewrites the engine's drain because "ours is better."** The engineer peer's primitive becomes stale, the SRE's reimplementation takes on the engine's bugs plus its own, and no one owns the drain path cleanly. The right action is a concrete bug report back to the engineer peer and a slower fallback pattern until the fix lands.
- **The mlops pipeline embeds release-gate logic directly.** The pipeline hard-codes "if ttft_p95 > X then hold" rules; the release-gate contract is no longer the source of truth; a threshold change requires a pipeline edit instead of a contract edit. The gate is bypassed by pipeline-local rules that drift.
- **The SRE sets eval thresholds.** A noisy judge produces frequent gate failures; the SRE lowers the eval threshold "temporarily" to keep releases flowing; the eval peer does not know; the gate silently becomes less strict. Threshold edits always go through the quarterly review (chapter 05 §7.1).
- **The eval team ships an eval change with no SRE awareness.** A new judge, a new eval set, a new rubric; the gate's pass rate changes overnight; the SRE sees a drop in release velocity and investigates. The change-control discipline (chapter 05 §7.2) catches this: eval changes are released through the contract-edit process.
- **Shared ServiceAccounts / shared credentials across peers.** The mlops pipeline, the release-gate service, and the engineer peer's engine orchestration all use a single `release-admin` ServiceAccount. A misconfiguration on any one side affects all three; audit cannot distinguish the sources. RBAC-split each peer's service into its own identity.
- **Shared vocabulary that drifts.** The mlops peer calls a cutover a "deployment"; the engineer peer calls it a "weight swap"; the SRE calls it a "release." A metric labelled `deployments_total` means different things to each. The quarterly cross-peer review reconciles vocabulary.
- **The release-gate becomes a performance venue.** Peers compete on "my gate pass rate is 92%, yours is 75%"; the metric displaces the quality of the predicates themselves. The report is for improvement, not comparison; the function's leadership reads the trend, not the ranking.

## 5. When the peer lines must still be read backwards

Boundary chapters sometimes over-fence. Three cases where the boundary bends:

### 5.1 The engine has a bug the engineer peer cannot immediately fix

- **Pattern.** The engine's drain primitive hangs on a specific long-context tail of requests; the engineer peer acknowledges the bug but needs a release cycle to fix.
- **SRE's role.** Document the workaround (longer drain period, fallback to full re-deploy), feed it into the `rb-blue-green-rollback` runbook, add a chaos experiment that exercises the condition, and track the fix in the engineer peer's backlog. The SRE does not fork the engine.

### 5.2 The eval team cannot produce a verdict fast enough for the release cadence

- **Pattern.** The eval pipeline runs overnight; releases are cut hourly; the gate holds on expired verdicts constantly.
- **SRE's role.** Document the mismatch, propose either a faster eval subset for per-release gating (the eval peer owns the design of the subset) or a reduced release cadence for the stream in question. The SRE does not run eval themselves.

### 5.3 The mlops pipeline needs a new gate predicate the SRE has not yet built

- **Pattern.** A new deployment target (e.g., edge deployment) needs a gate predicate the SRE's release-gate service does not yet implement.
- **SRE's role.** Add the predicate to the service, extend the verdict schema, revise the contract. The SRE does not tell mlops "just put the check in your pipeline"; the check lives in the gate service so the contract remains the source of truth.

In each case, the pattern is the same: the SRE identifies the cross-boundary signal, names the owning peer, holds the line on what the SRE does and does not own, and documents the workaround explicitly so it does not become the permanent shape.

## 6. The AI SRE's irreducible release-engineering surface

To restate what this module does own, now that the exclusions are explicit:

- **The canary, shadow, and blue-green patterns** (chapters 01, 03, 04).
- **The response-diffing engineering shape** (chapter 02 — the SRE wires the pipeline; the eval peer owns the judge).
- **The release-gate contract** (chapter 05).
- **The DORA four-keys computation and quarterly narrative** (chapter 06).
- **The release-time load-and-chaos pairing** (chapter 07).
- **The release-gate service as a dependency** — its API, its schema, its retention, its availability SLO.
- **The orchestration contract that invokes the engine's drain** (chapter 04 §5).
- **The audit trail for cutovers, rollbacks, pauses, and gate overrides.**
- **The runbooks for release-engineering operations**: `rb-blue-green-rollback`, `rb-canary-stuck-pause`, `rb-release-gate-service-down`, `rb-judge-pipeline-degraded-at-gate-time`, and the fixture back-feed loop with `mod-108` chapter 05.

Nothing above is co-owned. Everything above is accountable to the function.

## 7. The peer-boundary worksheet

Use this before a new release-engineering surface is added to the module's scope, and re-read it when a peer disagreement reaches the release-engineering lead:

1. **Does this work implement the inference engine's internals?** If yes, it belongs to the engineer peer.
2. **Does this work write the CI/CD pipeline?** If yes, it belongs to the mlops peer.
3. **Does this work author or tune eval sets, judge prompts, or quality rubrics?** If yes, it belongs to the eval-engineer peer.
4. **Does this work decide when to train or release a new candidate?** If yes, it belongs to the model-training / product owner.
5. **Does this work set retention / compliance policy on release records?** If yes, it belongs to `mod-110`'s compliance discipline.
6. **If none of 1–5 apply, is this work release orchestration or release contract?** If yes, it is this module's. Add it to §6.

A release-engineering program that can execute this worksheet every time its surface grows is a program whose boundary is honest. A program that cannot is a program that will drift into the peers' work and leave its own uncovered.

## Summary

- Release engineering on AI services is a joint practice with three peers — inference-engineer, mlops, eval-engineer — each owning distinct parts of the release path.
- The SRE's function does **not** implement the engine's internals, rewrite the CI/CD pipeline, or author eval methodology; it owns the orchestration contract, the release-gate contract, and the operational quality of the release path.
- Shared surfaces — engine drain primitive, release-gate contract, CI/CD-to-gate interface, judge pipeline, DORA report — have jointly authored artifacts and separately owned implementations.
- Anti-patterns named: SRE rewriting the engine, mlops hard-coding gate rules, SRE setting eval thresholds, eval shipping without SRE awareness, shared ServiceAccounts, drifting vocabulary, gate as a performance venue.
- When the boundary bends (engine bug, eval too slow, new mlops target), the SRE documents the workaround explicitly and routes the fix to the owning peer — the SRE does not fork.
- The function's irreducible release-engineering surface is canary / shadow / blue-green patterns, response-diffing engineering, release-gate contract, DORA, load-and-chaos pairing, release-gate service, orchestration contract, audit trail, and release-engineering runbooks.
- A six-point worksheet decides, for any new release-engineering work, whose surface it belongs to.

This closes module 111. The release engineering program it specifies is the practice that turns the module's chapters into a continuous delivery discipline for model updates, and the peer boundaries it draws are what let the function compound its work rather than compete with the peers whose expertise the release path depends on.
