# Peer Boundaries for AI Incident Response

## Why this chapter exists

Incident response is a composition discipline. The AI SRE owns the runbook library, the command structure, the retrospective cadence, the fixture back-feed, and the technical narrative at the regulatory handoff. The role does **not** own the serving-code fix, the model-weight change, the prompt-template rewrite, the vendor-contract clause, the safety-filter policy, the FinOps decision, or the regulator submission. Each of those sits with a named peer. A module on incident response that does not draw these lines explicitly invites the AI SRE to drift into work other roles are accountable for and to under-serve the composition work that no one else is accountable for.

This chapter closes the module by naming the peers, the handoff artifact that crosses each boundary, and the shared-artifact ownership patterns that keep the composition honest. It is a vocabulary chapter; the depth of each peer's work lives in that peer's own curriculum (`mod-101` chapter 02 is the full map).

## 1. The peer map for incident response

The eleven peers below are the ones an AI SRE will interact with in the first year of operating incident response. Each entry names the role, the surface they own in an incident, the AI SRE's handoff artifact, and the shared-ownership pattern.

### 1.1 `ai-infra-inference-engineer`

- **Owns:** the serving layer end-to-end — router, scheduler, KV-cache, batching, structured-output modes, streaming gateway, and the model-swap / weight-load code path.
- **Handoff artifact from the AI SRE:** the incident channel, the runbook state (chapter 01 §1), the confirm-checklist outcome, and the scope. For `rb-ttft-regression`, `rb-tpot-regression`, `rb-cache-hit-cliff`, `rb-structured-output-break`, `rb-streaming-stall`, `rb-model-swap-rollback`, and `rb-weight-corruption`, this peer is the OL by the time the runbook escalates.
- **Shared artifacts:** the fallback contract (`mod-107` chapter 02) is co-owned with vendor-risk; the serving-side eval canary gate is co-owned with `ai-eval-engineer`.

### 1.2 `ai-infra-performance`

- **Owns:** GPU driver, CUDA, kernels, NCCL, numeric format, cluster scheduling, node pools, and the fabric. The kernel-and-below layer.
- **Handoff artifact:** the hardware-level signal set (DCGM capture, XID codes, NCCL logs, step-time distributions). For `rb-gpu-driver-ecc-xid`, `rb-nccl-hang-straggler`, and the kernel class of `rb-preemption-spike`, this is the escalation peer.
- **Shared artifacts:** the chaos program for hardware faults (`mod-109`) is co-owned.

### 1.3 `ai-infra-platform`

- **Owns:** Kubernetes, the cluster-level scheduler config, node pools, the control plane, and the platform-level autoscaler. The cloud-provider interface.
- **Handoff artifact:** scope of the scheduler event, PDB / priority-class state, the node-pool configuration, the cloud-provider's status.
- **Shared artifacts:** the preemption / autoscaler chaos experiments are co-owned.

### 1.4 `ai-infra-ml-platform`

- **Owns:** the model registry, versioning, the artifact store, the model-promotion pipeline, and the progressive-rollout tooling.
- **Handoff artifact:** the registry entry, the version chain, the rollout-fraction state, the promotion gate's eval history. For `rb-model-swap-rollback` and `rb-weight-corruption`, this peer owns the rollback mechanics.
- **Shared artifacts:** the pre-promotion canary eval gate is co-owned with `ai-eval-engineer`.

### 1.5 `ai-infra-pipeline`

- **Owns:** data ingest, embedding pipelines, index build, chunking, and the retrieval index's operational surface.
- **Handoff artifact:** ingest lag, index freshness, embedding-model version, chunking configuration, index replica state. For `rb-retrieval-index-staleness` and `rb-embedding-drift`, this is the escalation peer.
- **Shared artifacts:** the embedding-model version-pin and the recall@k canary set are co-owned with `ai-eval-engineer`.

### 1.6 `ai-eval-engineer`

- **Owns:** canary eval sets, the judge-model pipeline, calibration sets, the eval-pipeline infrastructure, and the training-eval boundary.
- **Handoff artifact:** the eval version chain, the judge's calibration history, contamination-check results. For `rb-judge-model-drift`, `rb-eval-set-poisoning`, and any quality-signal interpretation question, this is the escalation peer.
- **Shared artifacts:** every canary-eval fixture from the back-feed cycle (chapter 05) is a joint artifact.

### 1.7 `AI Engineering` (the application / prompt layer)

- **Owns:** prompt templates, agent scaffolding, tool schemas, system prompts, feature flags, and the application-layer code that composes model calls into product features.
- **Handoff artifact:** the prompt-change history, the agent configuration, the tool schemas, the feature-flag state. For `rb-tool-call-cascade`, `rb-refusal-rate-spike`, `rb-hallucination-rate-spike` on prompt-caused classes, `rb-cache-hit-cliff` on template-caused classes, and `rb-structured-output-break` on schema-caused classes, this is the escalation peer.
- **Shared artifacts:** prompt / template change-management policy is co-owned with the AI SRE function.

### 1.8 `ai-risk-engineer`

- **Owns:** vendor-risk engineering, provider-contract interpretation, provider-level safety configuration, and the compliance-facing technical surface. This is the role chapter `mod-107` is largely about.
- **Handoff artifact:** the vendor dependency map, the fallback contract, the vendor-side status, the vendor-contract SLA interpretation. For `rb-provider-degraded-fallback`, provider-side silent-update events, and any trigger that crosses into contract or regulator surface, this peer is a direct collaborator.
- **Shared artifacts:** the regulator-facing technical narrative (chapter 06 §4) is drafted jointly.

### 1.9 Security

- **Owns:** authentication, authorisation, secret management, infrastructure security, supply chain, and the incident-response interface for security events.
- **Handoff artifact:** suspected account compromise, suspected supply-chain / artifact tampering, suspected prompt-injection with privilege escalation, eval-set-poisoning traces of malicious intent. For `rb-eval-set-poisoning` where tampering is suspected, and for any suspected model-weight tampering, security is the escalation peer.
- **Shared artifacts:** the artifact-preservation procedure (chapter 06 §5) is co-owned.

### 1.10 Legal / Compliance / PR

- **Owns:** the external communication, the regulator interface, the customer-contract interpretation, and the PR narrative.
- **Handoff artifact:** the technical narrative, the impact assessment, the artifact inventory, the classification map, the data-handling attestation (chapter 06 §4).
- **Shared artifacts:** the retention and legal-hold policy for AI-incident artifacts is co-owned with the AI SRE function.

### 1.11 FinOps

- **Owns:** cost allocation, budget policy, vendor-billing reconciliation, and the per-tenant / per-feature cost accountability.
- **Handoff artifact:** per-route hourly spend, per-tenant spend, feature-attribution data. For `rb-cost-blow-up` and the cost-adjacent surface of `rb-tool-call-cascade`, this is the escalation peer.
- **Shared artifacts:** per-route cost dashboards and per-session cost caps are co-owned.

### 1.12 Safety / Policy (where separate from Legal)

- **Owns:** the safety taxonomy, the moderation / filter configuration, the alignment-posture decisions, and the safety-review process.
- **Handoff artifact:** safety classifier output, flagged-output samples (through the controlled-access store), the filter configuration history. For `rb-toxic-output-blast` and `rb-refusal-rate-spike` where the regression suggests a filter change, this is the escalation peer.
- **Shared artifacts:** the safety-adjacent decision tree (chapter 02 §6) is co-owned with the AI SRE function.

### 1.13 Executive liaison

- **Owns:** the executive briefing cadence and the decisions that require executive authority (public statement, regulator engagement, large-scale customer compensation, cross-functional resource reallocation).
- **Handoff artifact:** the state-of-the-incident brief and the decision-request package. The exec liaison is paged by the IC per chapter 02's role list; the AI SRE is not the exec-briefing author in isolation.
- **Shared artifacts:** the severity ladder's SEV-1 criteria are co-owned.

## 2. The AI SRE's irreducible surface

The AI SRE's work in incident response is distinct from each of the peers' work. The irreducible surface — the work no peer owns and that the function therefore must — is:

- **The runbook library as a whole** (chapter 01). Individual runbooks are co-owned; the library's shape, the review cadence, and the CI enforcement are AI SRE's.
- **The command structure** (chapter 02). IC rotation, channel discipline, cadence, bridge-call hygiene, pre-committed decision trees.
- **The post-mortem process** (chapter 03). Template, review cycle, publication, co-authorship enforcement.
- **The pattern-matching practice against public incidents** (chapter 04). Dissection, synthesis, quarterly roll-up.
- **The fixture back-feed ledger** (chapter 05). Scheme, cross-incident de-duplication, measurement.
- **The regulatory-handoff discipline** (chapter 06). Trigger rules, artifact package shape, preservation protocol.

The composition of these six surfaces is what the role is hired to produce. Peers deliver in their own surfaces; the AI SRE ensures the surfaces compose into an incident lifecycle that a stakeholder outside the function can inspect.

## 3. Shared-artifact ownership patterns

Many of the artifacts this module produces are co-owned. Three ownership patterns recur and the function names them explicitly to avoid drift.

### 3.1 The AI SRE writes; the peer approves

- **Runbook per class.** The AI SRE drafts to the chapter 01 shape; the peer owner approves the technical specifics and the handoff routing.
- **Fixture record.** The AI SRE enters the ledger entry; the peer owner signs off on the system-of-record and the verification.
- **Technical narrative for legal.** The AI SRE drafts; legal / `ai-risk-engineer` review and finalise.

### 3.2 The peer writes; the AI SRE reviews for composition

- **Runbook mitigation menu for a serving-path runbook.** `ai-infra-inference-engineer` names the mitigations; the AI SRE reviews blast radius and reversibility framing.
- **Canary eval case.** `ai-eval-engineer` writes; the AI SRE confirms it maps to a chapter-03-taxonomy class and a specific past (or dissected) incident.

### 3.3 Joint-draft documents

- **Post-mortem with multi-category root cause.** Co-authored (chapter 03 §4.2).
- **Safety-decision tree.** AI SRE and Safety / Policy co-draft; legal reviews.
- **Fallback contract.** `ai-risk-engineer` and `ai-infra-inference-engineer` co-draft; AI SRE reviews for incident-response implications (`mod-107` chapter 02).

## 4. Anti-patterns at the boundary

Four anti-patterns are worth naming. The function's leadership watches for them:

### 4.1 The AI SRE as prompt owner

An AI SRE who starts fixing prompt bugs because the AI engineer's response is slow is doing two jobs and neither one well. The right response is to escalate the staffing gap, not to absorb the work.

### 4.2 The peer as retro author alone

A retro authored by the serving engineer alone on an incident that involved retrieval, evals, and the vendor will undercount. Chapter 03's co-authorship rule exists for this.

### 4.3 The AI SRE drafting the regulator communication

Chapter 06 is explicit. The role drafts the technical narrative; legal writes the submission.

### 4.4 The missing peer

Every runbook has a named peer owner. A runbook whose peer-owner field is blank is a runbook whose root-cause fix has no home. The library-authoring worksheet (chapter 01 §6) catches this.

## 5. Working across the boundary

Concrete practices keep the composition healthy:

- **Named rotations, not named people.** The AI SRE pages `ai-infra-inference-engineer-oncall`, not Sam. Handoffs survive vacations.
- **Written primary.** Every handoff that matters is in writing in the incident channel or the handoff document. Verbal is a supplement.
- **Explicit acknowledgement.** The incoming peer acknowledges the handoff in-channel. Silent handoffs are not handoffs.
- **Shared tooling.** The incident-response tooling, the runbook library, the fixture ledger, and the dashboards are shared across peers. The AI SRE's tooling is not a parallel universe; it is a shared surface.

## 6. The AI SRE's peer-boundary worksheet

1. **Enumerate the peers** who have an owning role for any part of the module's output. Compare to the list in §1.
2. **For every runbook in chapter 01, identify the peer owner.** Flag any runbook whose peer is missing or ambiguous.
3. **Identify the joint artifacts** (fallback contract, pre-promotion eval gate, safety decision tree, regulator-facing narrative) and the co-owner.
4. **Set a rotation-to-rotation handoff contract.** How is the peer paged, where, within what response window.
5. **Audit the retro corpus** for single-ownership on multi-surface incidents; those are co-authorship gaps.
6. **Publish the boundary map** in the function's documentation so new hires and peers see the same picture.

## Summary

- **Incident response composes across many peers.** `ai-infra-inference-engineer`, `ai-infra-performance`, `ai-infra-platform`, `ai-infra-ml-platform`, `ai-infra-pipeline`, `ai-eval-engineer`, `AI Engineering`, `ai-risk-engineer`, Security, Legal / PR, FinOps, Safety / Policy, and Executive liaison are the peers the AI SRE will touch in the first year.
- **The AI SRE's irreducible surface** is the library, the command structure, the retrospectives, the pattern-matching, the fixture back-feed, and the regulatory-handoff discipline. The composition of these surfaces is what the role is hired for.
- **Shared artifacts have explicit ownership patterns.** AI SRE drafts / peer approves; peer drafts / AI SRE reviews; joint draft. Naming the pattern prevents drift.
- **Four anti-patterns to watch for** — AI SRE as prompt owner, peer as retro author alone, AI SRE drafting regulator communication, and the missing peer on a runbook.
- **The boundary is maintained by named rotations, written primaries, explicit acknowledgements, and shared tooling.**

This chapter closes mod-108. The module's output composes forward into `mod-109`'s chaos engineering (which exercises every runbook and fixture this module produces), `mod-110`'s security-safety-compliance crossover (which interprets the regulatory handoff surface in more depth), `mod-111`'s release-engineering (which hardens the progressive-rollout gates that prevent many of the model-weight incidents here), and `mod-114`'s cross-functional reliability leadership (which gives the role the facilitation vocabulary to work across the peers named above at scale).
