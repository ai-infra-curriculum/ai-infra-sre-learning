# The Engine-Authoring Peer Boundary

## Why this chapter exists

Chapters 04, 05, and 06 taught *engine-knob fluency* — enough to triage a production outage against vLLM, SGLang, TensorRT-LLM / Triton, and NVIDIA Dynamo without paging the engine peer first. This chapter draws the line between where that fluency stops and where the `ai-infra-inference-engineer` peer at level 35 begins. It is deliberately explicit, because the line is easy to drift across in both directions — SREs who read a scheduler-loop patch once and start proposing kernel changes, and SREs who page the peer for every engine quirk — and the drift breaks the on-call shape authored in chapter 03.

The sibling curriculum that owns the authoring side is [`ai-infra-inference-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-inference-engineer-learning). Read its charter once so the boundary is reciprocal in your head.

## 1. The two-column contract

The same side-by-side structure from `mod-101` chapter 04, specialized to the engine surface.

| AI SRE owns (on-call depth) | `ai-infra-inference-engineer` owns (source depth) |
|---|---|
| Launch-flag configuration and audit | New launch flags, flag deprecation, flag semantics |
| Kubernetes manifests + Helm charts hosting the engine | Engine runtime code — scheduler loop, batcher, block manager |
| `config.pbtxt` + model-repository hygiene | New Triton backend implementations |
| Engine-version pinning + rollback mechanics | Engine release engineering (cutting a new vLLM / SGLang version) |
| Build-pipeline wiring for TRT-LLM engines | `trtllm-build` internals, plugin authoring |
| Dynamo pool sizing + topology | Dynamo routing-policy source, KV-transfer internals |
| Observability wiring (`/metrics`, trace propagation) | Engine's internal metric authoring, new metric surfaces |
| Reversible knob changes during incident response | Source-level fixes, hotfix cherry-picks, version bumps |
| Workload-aware tuning parameter sweeps | Kernel, CUDA, PTX, SASS authoring |
| SLO contracts the engine must meet | Engine performance-target discovery |

Every row on the left lands in your queue at an on-call page; every row on the right lands in your queue as a ticket *you file*, not a diagnosis you perform.

## 2. The three canonical hand-offs

Three situations where the hand-off from operator to peer is the right move. Recognize them.

### Hand-off 1 — Suspected scheduler bug

Symptom: engine is admitting requests in a pattern that produces head-of-line blocking under long-context load, chunked prefill is enabled, max-batched-tokens is tuned, and the pattern persists. The operator has exhausted the reversible knob levers in chapters 04–06 and the SLO is still burning.

**Correct move.** Capture the metric trace (engine metrics window, request-arrival timing, decode-step timing), capture the launch-flag set and engine version, open a ticket upstream, and roll back to the prior engine version while the peer investigates. The peer may confirm a scheduler regression and produce either a patch or a confirmed workaround.

**Incorrect move.** Reading the scheduler source, writing a candidate fix, submitting a pull request. That work is in the peer's charter; your charter is restoring the SLO.

### Hand-off 2 — Suspected kernel regression

Symptom: engine version was upgraded; TPOT regressed; no launch-flag change; no model change; the regression lands within a single engine release.

**Correct move.** Capture the before-and-after telemetry (TPOT percentiles, GPU util, DCGM profile), capture both engine versions, open a ticket co-signed by the kernel SME (the `ai-infra-performance` peer at level 35 — Nsight / CUDA depth) and the engine peer. Roll back to the prior version in the meantime.

**Incorrect move.** Opening Nsight-Compute, reading SASS, proposing a kernel-authoring change. That work is in the performance peer's charter.

### Hand-off 3 — Feature-gap triage

Symptom: a product team wants to ship a workload the current engine configuration cannot support — a context length the engine was not built for, a quantization scheme not implemented, a new scheduling policy not expressed by available knobs.

**Correct move.** Confirm it is a feature gap (not a knob you missed), file an engine ticket with the use case and the requested behaviour, and offer an operator-side workaround in the meantime (admission clamp, workload split, fallback route). The peer scopes the work.

**Incorrect move.** Implementing the feature in a fork and running the fork in production. Even if the patch is small, the fork becomes a reliability surface you then own forever.

## 3. The ticket template

The ticket is the artifact the hand-off is executed with. Treat it as a reliability artifact. Minimum template:

- **Title.** One-line symptom (e.g., "vLLM 0.7.x TPOT regression on 70B model, chunked prefill enabled, no launch-flag change").
- **Engine.** Name, exact version, commit SHA if available.
- **Deployment surface.** Which pool, how many replicas, GPU shape.
- **Launch flags.** Exact flag set in use.
- **Workload profile.** Token throughput, prompt / generation length distribution, concurrent-sequence distribution.
- **Symptom.** SLO affected, metric trace, time range. Attach graphs.
- **What was tried.** Reversible knob levers attempted, each with its result.
- **Current workaround.** What you have done to hold the SLO (rollback, admission clamp, fallback route).
- **Severity.** Mapping to your error-budget band from chapter 02.

The template is boring on purpose. The peer should be able to pick up the work without a 30-minute meeting.

## 4. The reverse direction — what the engine peer routes to you

For completeness, the peer's side of the handoff. Your charter should accept these cases from them:

- **Engine release rollout.** The peer cuts a new engine version; you own the rollout plan, canary, progressive deploy, and SLO gate (`mod-111`).
- **Engine configuration recommendations.** The peer publishes a recommended flag set for a new model; you operationalize, validate against production, and feed back the measured impact.
- **Engine regression incidents.** The peer produces a hotfix; you run the hotfix through the rollout pipeline and catch any regression against your SLO stack.
- **Capacity-planning signals from the engine side.** The peer characterizes the engine's throughput ceiling under the new configuration; you factor into fleet sizing (`mod-106`).

A healthy relationship has both sides of the arrow populated. If you only receive hand-offs from the peer and never send them any, you are probably absorbing work that is theirs.

## 5. Shared sources of truth

Both sides read the same documents:

- Engine release notes for every version pinned in production.
- Engine issue trackers for open bugs affecting your pools.
- Engine benchmark publications (NVIDIA Dynamo blog, vLLM blog, SGLang release notes) for the throughput characterizations that feed capacity planning.
- Shared metric endpoints — `/metrics` — exposed by every engine.

Keep a short "engines in production" page in your runbook library listing, for each pool: engine name, pinned version, commit SHA, upstream release-notes URL, list of launch flags, link to the peer's owning channel. This page is a cross-team artifact; both sides should recognize themselves in it.

## 6. Signals the boundary is drifting

### You are drifting into the peer's work if

- You have an in-flight patch to the engine source and you are not on the peer's team.
- A fork of the engine is running in production.
- You are debugging CUDA kernels in your on-call shift.
- You are reading SASS to triage a regression.

### The peer is drifting into your work if

- The peer is tuning launch flags on your production pools directly.
- The peer is editing `config.pbtxt` or Kubernetes manifests in your repos.
- The peer is driving incident response on an outage with no engine-source root cause.

Both drifts are solvable with a brief conversation at the next cross-team sync. The reliability charter and the deferral map from `mod-101` are the artifacts that resolve it; this chapter's two-column contract specializes them to the engine surface.

## Summary

- **Two columns.** The AI SRE owns configuration, wiring, rollout, SLO contract, and reversible operational knobs. The inference-engine peer owns engine source, release engineering, kernel authoring, and feature development.
- **Three canonical hand-offs.** Suspected scheduler bug, suspected kernel regression, feature gap. In each, the operator captures the artifact, files the ticket, and holds the SLO with a workaround; the peer owns the fix.
- **Ticket template.** Standardized, boring-on-purpose, nine required sections. The point is to make the hand-off executable without a meeting.
- **Reverse direction.** The peer routes rollouts, configuration recommendations, hotfixes, and capacity signals to you. A healthy relationship uses both arrows.
- **Shared sources of truth.** Engine release notes, issue trackers, benchmark publications, the "engines in production" runbook page.

This chapter closes the module. You now have the vocabulary (chapter 01), the error-budget policy that uses it (chapter 02), the on-call rotation that enforces the policy (chapter 03), engine-knob fluency for the three serving-engine families on-call will see (chapters 04–06), and the explicit boundary to the engine-authoring peer (this chapter). The exercises apply all six against a team context you pick in exercise 01 and carry through.
