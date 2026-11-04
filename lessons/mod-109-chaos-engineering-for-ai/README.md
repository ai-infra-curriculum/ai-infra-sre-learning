# mod-109-chaos-engineering-for-ai: Chaos Engineering for AI Workloads

**Estimated effort:** 16 hours

An AI SRE operates on a fleet whose failure modes include every classical failure (pod kill, network loss, disk latency) *and* a parallel AI-specific set (GPU preemption under in-flight stream, provider timeout with half-delivered response, prefix-cache invalidation, degraded-mode routing, agent tool-call cascade, retrieval index drop, judge-model unavailability). The classical chaos canon — Principles of Chaos Engineering, Chaos Mesh / LitmusChaos / Gremlin / AWS Fault Injection Service — gives shape and tooling; it does not give the AI-specific catalogue, the AI-amplified blast-radius controls, or the fallback-contract chaos suite. This module writes those, as a continuous program with scheduled game-days, auto-halt wired to the SLO stack, and fixture back-feed to the runbook library from `mod-108`.

## Learning objectives

- Ground the module in the Principles of Chaos Engineering — steady-state hypothesis, blast-radius discipline, production-first bias, automated continuous experiments — and extend each for AI-specific workloads
- Deploy a chaos program with Chaos Mesh / LitmusChaos / Gremlin / AWS Fault Injection Service — pod chaos, network chaos, I/O chaos, DNS chaos, time chaos — in the K8s substrate the AI fleet runs on
- Author AI-specific chaos experiments — GPU eviction mid-stream, model-provider timeout injection, prefix-cache poisoning / invalidation, degraded-mode routing drills, agent-tool-call fault injection, worker-crash under in-flight stream, graceful-preemption under KV pressure, judge-model unavailability, retrieval-index drop
- Run the chaos program as a continuous practice — scheduled game-days, Principal / Staff-engineer involvement, blast-radius controls, auto-halt on SLO burn, learnings-to-runbook feedback loop
- Chaos-test the provider-fallback contract from `mod-107` — inject upstream failures and measure MTTD / MTTR / fallback-hit-rate; prove the contract holds under realistic failure modes
- Cite the boundaries — chaos targets reliability behaviour this role owns; adversarial robustness testing (prompt-injection, model-extraction, jailbreak) is at the `ai-infra-security` peer, and model-eval robustness testing is at `ai-eval-engineer` / `model-evaluation-engineer` peers

## Chapter index

- [`01-principles-of-chaos-adapted-to-ai.md`](./01-principles-of-chaos-adapted-to-ai.md) — The five canonical principles (steady state, real-world events, production first, automate continuously, minimise blast radius) and the AI-specific extension for each: three parallel metric families (serving, quality, cost); the AI fault catalogue overlay; the tier-1 / tier-2 production split; the cadence demanded by the AI service's own change rate; the AI-amplified blast-radius caps; and a sixth AI-only principle — every chaos finding becomes a permanent fixture.
- [`02-chaos-tooling-in-the-k8s-ai-substrate.md`](./02-chaos-tooling-in-the-k8s-ai-substrate.md) — The five fault classes Kubernetes substrates expose (pod, network, I/O, time, stress); the AI-specific gap (no first-class GPU-chaos primitive) and how to compose it; the four tools (Chaos Mesh, LitmusChaos, Gremlin, AWS Fault Injection Service) and what each is really for; the egress-shaping pattern for provider / judge / tool / retrieval chaos; installation, RBAC, GitOps, and dry-run hygiene; a substrate-readiness checklist.
- [`03-ai-specific-chaos-experiments.md`](./03-ai-specific-chaos-experiments.md) — The nine-experiment floor (GPU eviction mid-stream, provider-timeout injection, prompt-cache invalidation / poisoning, degraded-mode routing drill, agent tool-call fault, worker crash under stream, KV-pressure graceful preemption, judge-model unavailability, retrieval-index drop); each with production fault model, three-family hypothesis, mechanism, blast-radius controls, runbook ties, expected observations, and fixture back-feed hooks; the no-chaos list.
- [`04-provider-fallback-contract-chaos.md`](./04-provider-fallback-contract-chaos.md) — The six provider-failure shapes (5xx storm, 429 sustained, TCP stall, streaming stall mid-response, degraded-200, region-red-global-green) and the chaos suite that verifies the `mod-107` chapter 02 fallback contract against each; the three metrics (MTTD, MTTR, fallback-hit-rate) and their budgets; the suite as a release gate; the predictable bug catalogue this program catches.
- [`05-continuous-chaos-program-and-game-days.md`](./05-continuous-chaos-program-and-game-days.md) — The four-tier cadence (per-change / daily / weekly / quarterly); auto-halt wiring with three independent alarms (serving / quality / cost); the blast-radius policy document (signed, version-controlled, reviewed quarterly); the game-day shape with Principal / Staff-engineer participation and SME roles; the maturity staircase (levels 0–4); the pitfalls that defeat first-generation programs.
- [`06-peer-boundaries-for-chaos.md`](./06-peer-boundaries-for-chaos.md) — The three disciplines on the same substrate (AI SRE chaos, `ai-infra-security` adversarial robustness, `ai-eval-engineer` model-evaluation robustness); what the AI SRE chaos program explicitly does not do; the shared surfaces (canary evals, safety-classifier halt signals, fault-injection tooling, cross-discipline game-days); the anti-patterns; the AI SRE's irreducible chaos surface.

## Structure

- `01-…md` … `06-…md`: lecture chapters.
- `exercises/`: per-exercise prompts (principles adoption, tooling install + first experiment, GPU / worker-crash experiments, provider-timeout / degraded-mode experiments, cache / retrieval experiments, continuous-practice game-day plan).
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## How this module composes with the track

- **Input from `mod-101`.** The AI SRE's generalist-with-routing-competence framing in chapter 01 of that module is the premise under which chapter 06 draws the peer boundaries.
- **Input from `mod-102`.** Classical SRE discipline — SLOs as contracts, error budgets, non-abstract large-systems design — is the ancestry the chaos principles extend in chapter 01.
- **Input from `mod-103`.** The SLO-stack provides the steady-state baselines this module's experiments test against; auto-halt (chapter 05 §2) reads burn-rate alerts from this stack.
- **Input from `mod-104`.** Every detection signal cited in the chapter 03 catalogue is grounded in a `mod-104` observability artifact. A chaos experiment whose expected observation is not in the dashboard is uninterpretable.
- **Input from `mod-105`.** The non-deterministic failure taxonomy is the vocabulary chapter 03's experiments trace to; the three-family steady-state hypothesis (chapter 01 §2) reads this module's quality-signal set from `mod-105`.
- **Input from `mod-106`.** Capacity planning and load-testing surfaces are the home of the load generators this module's experiments depend on (KV-pressure, long-stream canaries, preemption triggers).
- **Input from `mod-107`.** The vendor dependency map, fallback contract, and provider-status ingestion are the subject of chapter 04's contract-chaos suite.
- **Input from `mod-108`.** The runbook library (chapter 01), the fixture back-feed cycle (chapter 05), and the incident command structure (chapter 02) are what the chaos program verifies, feeds, and borrows the game-day command cadence from.
- **Output to `mod-110`.** The security-safety-compliance crossover reads this module's peer-boundary chapter (chapter 06) as the input for where reliability chaos ends and `ai-infra-security`'s adversarial program begins.
- **Output to `mod-111`.** Release engineering uses the contract-chaos suite (chapter 04) as a release gate for router / serving changes.
- **Output to `mod-113`.** DR / BCP for AI consumes this module's game-day shape (chapter 05 §4) as the pattern for its own full-scale drills.
- **Output to `mod-114`.** The cross-functional leadership module operates at the level this module's peer-boundary chapter sets.
