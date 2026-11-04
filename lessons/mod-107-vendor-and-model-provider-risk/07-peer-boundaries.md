# Peer Boundaries — Where Vendor-Risk Engineering Stops

## Why this chapter exists

Vendor-and-model-provider risk sits at the junction of several deeper disciplines. The AI SRE owns the reliability framing: the dependency map, the fallback contract, the status-page pipeline, the SLA/SLO math, and the vendor-eval scorecard. Those artifacts intersect with work that other roles own end-to-end — the cost-performance router build, the legal and contractual SLA negotiation, the security review, the base inference-engine selection, and the broader commercial relationship management. Pretending the AI SRE owns all of them is role creep that produces worse outcomes in every lane.

This chapter draws the boundaries the earlier chapters only implied. For each peer discipline, it names what they own, what the AI SRE consumes from them, when to route an incident or decision to them, and — explicitly — what the AI SRE *does not do* even when the problem is adjacent.

## 1. The peer boundaries that matter here

### 1.1 The `ai-infra-inference-engineer` peer

**Owns.** The in-house serving engine when one is run (vLLM / SGLang / TensorRT-LLM / TGI / Triton); the cost-performance router implementation (the actual LiteLLM / Portkey / bespoke code and its performance tuning); the engine- and endpoint-level knobs that affect per-request cost (quantisation, speculative decoding, context-window policies, prefix-cache sizing); the build-vs-buy analysis for self-hosting; the ongoing benchmark program that produces per-provider / per-model cost and latency data.

**What vendor-risk consumes from them.**

- The current cost and latency profile per (provider, model) combination, updated as providers change pricing or models.
- The build-vs-buy conclusions that inform self-host decisions (chapter 06 §3).
- The router implementation that operationalises the fallback contract (chapter 02 §5).
- The engine-level metric surface that exposes retry, fallback, and provider-switch events for observability (`mod-104`).

**When to route to them.**

- The symptom is a router mis-behaviour: a fallback did not fire when it should have; semantic breakage across a swap (idempotency, streaming); schema inconsistency between providers.
- A decision about which engine to run or whether to self-host is on the table.
- Per-request cost is the dominant concern and the question is "which model would be cheaper here" — the inference-engineer peer owns the cost-aware routing logic.
- Model-quality routing decisions where the input is "run a benchmark to decide which model is strongest on this traffic shape."

**What vendor-risk does not do.**

- Write or tune the router / gateway code. The contract (chapter 02 §1) is the AI SRE's artifact; the code that implements it is the inference-engineer peer's artifact.
- Benchmark providers for cost-per-token or throughput. That data is consumed, not produced.
- Design the cost-aware routing policy that chooses between tier-A alternates based on price. The AI SRE's policy addresses reliability (break glass on degradation); the cost-optimised policy is a separate layer by the inference-engineer peer that composes with the reliability layer.
- Own the self-hosted fleet operation (chapters 04/05/06 of `mod-106` and the `ai-infra-inference-engineer` track cover this).

### 1.2 Legal and vendor-management

**Owns.** Contract terms with every upstream vendor — the MSA, the data-processing agreement, the BAA for HIPAA-covered workloads, the specific SLA language (percentage, measurement method, remedies, service credits, carve-outs), the term, the price, the auto-renewal clauses, the security addenda, the AI-specific clauses (training-use, output ownership, indemnification). Commercial negotiation. Renewal calendar. Payments. Procurement workflow.

**What vendor-risk consumes from them.**

- The exact SLA language per vendor product for the dependency map (chapter 01 §1.5).
- The contract-residency clauses that inform the fallback-contract's hard-no-op cases (chapter 02 §4.5, chapter 05 §5).
- The expiry and renewal dates for reservations and commitments (chapter 06 §5).
- The enforceability of service credits when an incident breaches the SLA.

**What vendor-risk delivers to them.**

- The required SLA percentage for a given downstream SLO (chapter 05 §6), so the negotiation has a concrete target.
- The exit-cost estimate per vendor (chapter 06 §5), as negotiation leverage.
- The composed-ceiling math showing which SLA improvements would matter and which would not.

**When to route to them.**

- Any question about *what the contract says*. The published SLA on a vendor's web page is a public ceiling; the signed contract may differ.
- Any question about pricing negotiation, renewal, or exit of a reservation.
- Any question about which clauses can be amended and which cannot.
- Breach-response and credit-claim process.

**What vendor-risk does not do.**

- Negotiate the contract. The AI SRE provides the technical input and leaves the negotiation to the owners.
- Interpret ambiguous contract language on their own. Pull legal in.
- Make the exit decision on commercial grounds. The engineering view of exit cost is one input; commercial, strategic, and relationship factors are others.
- Communicate the reliability problem to the vendor through unofficial channels when the official process goes through vendor-management.

### 1.3 Security and compliance

**Owns.** Vendor security review (SOC 2 / ISO 27001 / FedRAMP / HIPAA / PCI / regional certifications); the data-flow review that classifies data traversing a vendor; the training-use posture; the vulnerability- and incident-disclosure terms; the privacy-impact assessment under GDPR / CCPA / FADP; the red-team / model-safety assessment for new providers.

**What vendor-risk consumes from them.**

- The approved-vendor list for each data classification (public, internal, restricted, regulated).
- The compliance labels on each vendor (certifications held, BAA status, data-zone commitments).
- The security-response terms that factor into the fallback contract's residency hard-fail cases.

**When to route to them.**

- A new vendor is being evaluated. Security review precedes production use.
- A vendor's security posture materially changes (certification lapsed, breach disclosed, ownership change).
- A customer or regulator asks for the end-to-end data-flow diagram covering upstream AI vendors.

**What vendor-risk does not do.**

- Perform the security review. The AI SRE consumes the result.
- Approve a vendor for a new data classification.
- Author the data-flow diagrams that satisfy a regulator; security + legal own them.

### 1.4 Finance and FinOps

**Owns.** Committed-spend budgeting across all vendors; revenue-share of AI spend; margin analysis; cost allocation across internal business units (showback / chargeback); the FinOps dashboards that track AI spend against forecast.

**What vendor-risk consumes from them.**

- The budgetary envelope that bounds reservation decisions.
- The chargeback method that decides whether an incident's error-budget burn translates to a cost allocation.

**What vendor-risk delivers to them.**

- The forecast deltas when a fallback decision shifts traffic between vendors.
- The cost of redundancy (active-active + reservation on two vendors) as input to the trade-off.

**What vendor-risk does not do.**

- Own the cost-allocation plumbing. FinOps / platform-engineering build and run that.
- Decide margin targets.

### 1.5 The `ai-infra-engineer` peer at level 20 (platform)

**Owns.** The platform on which the router, the status ingestor, the observability plane, and the services themselves run. Cluster operations, secret management, ingress, service mesh, egress policies, DNS, network observability (eBPF-level), and the base-service template every team inherits.

**What vendor-risk consumes from them.**

- The egress-policy plane that enforces which workloads can call which vendor endpoints. (A route in the dependency map that is not permitted by egress policy is a mis-mapped route.)
- The secret-management plane that scopes API keys per service / per workspace / per environment.
- The service-mesh capabilities that implement retries, timeouts, and circuit-breakers for the router layer.

**When to route to them.**

- The ingestor cannot reach a status page from inside the cluster — egress-policy, DNS, or TLS trust-chain issue.
- A route's API key is leaking to a logs plane — secret-handling issue.
- The service mesh is interfering with provider-SDK behaviour (e.g., gRPC keepalive clashing with a vendor's timeout).

### 1.6 The `ai-infra-pipeline` peer

**Owns.** Data and config pipelines (`mod-112`'s subject): the pipelines that produce embeddings, that update retrieval indexes, that load prompt catalogues, that ship fine-tunes to vendors. Vendor-side artifacts (uploaded files for fine-tuning, assistants, vector-store contents at OpenAI, custom-model artifacts at Bedrock or Vertex) live in their domain.

**What vendor-risk consumes from them.**

- The list of vendor-side artifacts the service depends on (fine-tunes, vector stores, assistants, custom models), each of which is a dependency row (chapter 01 §3) with its own lifecycle.
- The migration cost estimate for moving those artifacts between vendors (chapter 06 §5).

## 2. The AI SRE's role in a multi-peer incident

A real incident often crosses boundaries. A sample flow for a sustained 5xx storm on `openai.chat.completions`:

1. **Ingestor fires** (chapter 04). Breaker opens. Router auto-fails to the active-active alternate.
2. **AI SRE on-call takes the incident.** Confirms the fallback is holding. Notes the error-budget burn against the provider-degradation reserve (chapter 05 §4).
3. **If the fallback shows stress** (quota approaching ceiling, quality regression on the alternate), the AI SRE engages the inference-engineer peer: can we increase batch efficiency? Is there a configuration knob?
4. **If the incident lasts long enough** to justify a vendor dialogue, vendor-management is engaged. They open the official escalation; the AI SRE provides the trace-ids, request-ids, and error-rate graphs.
5. **If a legal-exposure question arises** (a user-data event, a compliance question about where traffic is now served from the fallback), security and legal are engaged.
6. **Post-incident**, the AI SRE owns the vendor-risk-specific action items (does the contract need the fallback added? does the dependency map need an update? does the scorecard get a new reliability row?). Other peers own their own action items.

The AI SRE's lane is clear: **coordination across peer boundaries during the incident, ownership of the vendor-risk artifacts after**.

## 3. Shared artifacts

Several artifacts are *co-authored* across peers. The AI SRE is the primary maintainer but the other peers contribute.

### 3.1 The dependency map (chapter 01)

- AI SRE owns the schema, structure, and reliability columns.
- Service teams own the per-route entries.
- Legal / vendor-management provides the SLA language and reservation metadata.
- Security provides the compliance labels.
- Pipeline peer provides the vendor-side-artifact rows.

### 3.2 The fallback contract (chapter 02)

- AI SRE owns the contract document.
- Service team owns the policy's user-facing implications (what degraded mode means in the UI).
- Inference-engineer peer owns the implementation that enforces the contract.
- Legal confirms the residency hard-fail cases are correct.

### 3.3 The error-budget table (chapter 05)

- AI SRE owns the composition math.
- Service team owns the SLO target.
- Legal / vendor-management provides the SLA numbers.
- FinOps consumes the budget-burn-to-cost translation.

### 3.4 The vendor-eval scorecard (chapter 06)

- AI SRE owns the scorecard shape and the reliability / exit-cost sections.
- Inference-engineer peer provides cost / quality / performance rows.
- Security provides the compliance section.
- Legal provides the contract-terms summary.

A scorecard that is only the AI SRE's work is incomplete; a scorecard that nobody owns is stale.

## 4. What is explicitly out of scope for this track

Several topics come up in vendor-risk conversations and are *not* in the AI SRE's remit:

- **The cost-aware router.** The decision "which cheaper model do I use when the SLO allows it" is the inference-engineer peer's. The AI SRE's role is to specify the SLO quality floor so the cost router does not drive quality off it.
- **The commercial negotiation.** The AI SRE provides the required SLA number; legal and vendor-management negotiate how to get it.
- **The vendor-security review.** Security owns it; the AI SRE consumes the output.
- **The decision to self-host.** The inference-engineer peer owns the build-vs-buy analysis; the AI SRE provides the reliability constraints (SLA, residency) that bound the solution space.
- **The red-team / model-safety review of a new vendor.** The ai-safety / model-safety function owns this; the AI SRE references their conclusion in the scorecard.

## 5. The AI SRE's peer-boundary worksheet

1. **Keep the artifact ownership explicit.** Every artifact (map, contract, table, scorecard) has a primary owner; the other peers are contributors, not co-equal editors.
2. **Route early, not late.** When a decision crosses into a peer's lane (commercial, security, engine, pipeline), bring them in at the start of the thread, not when the engineering question has calcified.
3. **Produce one artifact per lane.** The AI SRE's output to legal is a one-pager with the required SLA number; the output to the inference-engineer peer is the fallback-contract specification; the output to security is the vendor list that needs review. The lane's owner consumes exactly what they need.
4. **Resist re-doing a peer's work.** Benchmarks, security reviews, commercial negotiations — each is deep in its own right. The AI SRE's contribution is correct framing, not duplication.
5. **Own the integration.** The AI SRE's job that no other peer does is the *composition* — turning the peer-produced inputs into a dependency map, a fallback contract, an error-budget table, and a scorecard that are mutually consistent.

## Summary

- **Vendor-risk engineering is a composition role**, not a universal-ownership role. Each adjacent discipline owns deep work; the AI SRE composes their outputs into the reliability artifacts.
- **Inference-engineer owns the router implementation and cost-aware routing**; the AI SRE owns the fallback contract that the router implements.
- **Legal and vendor-management own the contract**; the AI SRE owns the SLA-to-SLO math that tells them what to negotiate for.
- **Security owns the vendor security review**; the AI SRE consumes the result in the scorecard.
- **FinOps owns cost allocation**; the AI SRE owns the forecast deltas that feed it.
- **Shared artifacts have a primary owner and named contributors.** Ambiguous ownership produces stale artifacts.
- **The AI SRE's irreducible job is composition and coordination**: producing a dependency map, a fallback contract, an SLA/SLO table, and a scorecard that are mutually consistent and backed by the right peers.

With the boundaries drawn, the module concludes. The exercises operationalise each chapter into a concrete artifact an AI SRE should be able to produce for a real workload.
