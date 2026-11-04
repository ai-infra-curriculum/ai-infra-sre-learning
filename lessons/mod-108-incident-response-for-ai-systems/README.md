# mod-108-incident-response-for-ai-systems: Incident Response, Runbooks, and Post-Mortem Culture for AI Systems

**Estimated effort:** 18 hours

An AI SRE's job is not only to detect that a model-backed service is burning its error budget; it is to **run the incident** that follows. The classical SRE toolkit — ICS roles, blameless post-mortems, runbook libraries, back-feed to the SLO stack — carries over almost unchanged in shape but needs AI-specific content in every slot. The runbook library covers failure classes classical SRE does not teach. The command structure pulls in a wider SME bench and a higher-cadence legal / safety interface. The post-mortem template adds AI-specific root-cause categories and mandates fixture back-feed. The pattern-matching practice extends to the industry's public corpus. The regulatory handoff is a real surface this role is expected to execute cleanly without owning the formal communication. This module puts each of those pieces in place.

## Learning objectives

- Author the AI-reliability runbook library — the top-twenty AI-specific runbooks: TTFT regression, TPOT regression, cache-hit cliff, preemption spike, hallucination-rate spike, toxic-output blast, refusal-rate spike, tool-call cascade, provider-degraded fallback, GPU driver / ECC / XID incident, NCCL hang / slow-rank straggler, model-weight corruption, model-swap-rollback, retrieval-index staleness, embedding-drift incident, judge-model drift, eval-set poisoning, structured-output conformance break, streaming-stall, cost-blow-up
- Operate incident command — IC vs. OL vs. SME roles adapted to AI systems, bridge-call hygiene, incident-channel discipline, stakeholder-communication cadence, legal / compliance / PR coordination hooks for AI-safety-adjacent incidents
- Run blameless post-mortems — the Google SRE post-mortem template adapted with AI-specific root-cause categories (model-weight, kernel, scheduler, prompt / config, retrieval, provider, agent-cascade, eval-drift), mandatory permanent-regression-fixture back-feed, cross-peer co-authorship when the root cause spans ownership
- Pattern-match against real public post-mortems — read, dissect, and synthesise AI-specific incidents published by OpenAI / Anthropic / Google / Microsoft / frontier-lab engineering blogs — and extract the shape that this role is hired to prevent
- Build regression-fixture back-feed discipline — every post-mortem produces at least one permanent fixture (alert, canary case, eval-set entry, load-test shape, chaos experiment, runbook update) that would have caught the incident; close the loop in the SLO stack
- Hand off to legal / compliance / regulator-facing teams when an incident crosses that line — EU AI Act Article 72 post-market-monitoring report, FDA PCCP triggered change, SR 11-7 model-risk-management notification; this role contributes the AI-specific telemetry and defers formal reporting to ai-risk-engineer + legal

## Chapter index

- [`01-top-twenty-ai-runbook-library.md`](./01-top-twenty-ai-runbook-library.md) — The seven-section runbook shape (header, detection signals, confirm checklist, mitigation menu, handoff, user communication, fixture back-feed); the twenty AI-specific runbooks that the on-call must have on day one; keeping the library honest with review cadence, incident back-feed, and chaos verification.
- [`02-incident-command-for-ai-systems.md`](./02-incident-command-for-ai-systems.md) — IC / OL / SME / Comms / Scribe / exec-legal-PR liaison roles; incident-channel discipline, bridge-call hygiene, posting cadence; stakeholder communication protocol including the safety-adjacent pre-committed decision tree; handoffs across rotations and timezones; AI-specific communication traps.
- [`03-blameless-post-mortems-for-ai.md`](./03-blameless-post-mortems-for-ai.md) — The template adapted with AI-specific root-cause categories (model-weight, kernel, scheduler, prompt / config, retrieval, provider, agent-cascade, eval-drift); co-authorship for multi-surface incidents; drafting discipline and the two-day draft, five-day submission schedule; the review cycle (peer-SME, AI SRE function, legal where triggered).
- [`04-public-post-mortem-pattern-matching.md`](./04-public-post-mortem-pattern-matching.md) — The dissection template; the public corpus (frontier-lab blogs, cloud-provider post-mortems, incident catalogues like AI Incident Database and OECD.AI Monitor); recurring patterns (observability catches the wrong layer first, progressive-rollout as the dominant preventive control, vendor transparency as a correlated signal, safety incidents behaving differently, shared-substrate correlation); the quarterly synthesis.
- [`05-regression-fixture-back-feed.md`](./05-regression-fixture-back-feed.md) — The six fixture classes (alert, canary eval case, eval-set entry, load-test shape, chaos experiment, runbook update); the Candidate → Owned → Implemented → Verified → Active → Measured cycle; the fixture-ledger schema; closing the SLO-stack loop; the AI-specific traps (eval-set contamination, judge-drift invalidation, prompt-cache contamination).
- [`06-legal-compliance-handoff-for-ai-incidents.md`](./06-legal-compliance-handoff-for-ai-incidents.md) — The handoff premise (defer formal communication, produce the technical narrative, preserve everything); EU AI Act Art. 72 / 73, FDA PCCP, and SR 11-7 as the three most likely frameworks; triggers that obligate handoff; the artifact package (technical narrative, impact assessment, artifact inventory, classification map, data-handling attestation); preservation discipline; post-handoff follow-through.
- [`07-peer-boundaries.md`](./07-peer-boundaries.md) — The peer map (`ai-infra-inference-engineer`, `ai-infra-performance`, `ai-infra-platform`, `ai-infra-ml-platform`, `ai-infra-pipeline`, `ai-eval-engineer`, `AI Engineering`, `ai-risk-engineer`, Security, Legal / PR, FinOps, Safety / Policy, Executive liaison); the AI SRE's irreducible surface; shared-artifact ownership patterns (AI SRE drafts / peer approves, peer drafts / AI SRE reviews, joint draft); anti-patterns at the boundary.

## Structure

- `01-…md` … `07-…md`: lecture chapters.
- `exercises/`: per-exercise prompts (runbook authoring, incident command, post-mortem template, public post-mortem dissection, fixture back-feed, legal / compliance handoff).
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## How this module composes with the track

- **Input from `mod-101`.** The AI SRE's generalist-with-routing-competence framing in chapter 01 of that module is the premise under which chapter 07 draws the peer boundaries.
- **Input from `mod-103`.** The SLI / SLO contract is the trigger for every incident this module runs; chapter 05's fixture back-feed closes the SLO-stack loop.
- **Input from `mod-104`.** The observability stack produces every detection signal cited by the runbooks in chapter 01 and the post-mortem template in chapter 03. A runbook that references a signal the dashboard does not have is a broken runbook.
- **Input from `mod-105`.** The AI-specific failure taxonomy is the recognise-and-route vocabulary; this module adds the recognise-and-mitigate layer by name.
- **Input from `mod-106`.** Capacity planning and load-testing surfaces are the home of chapter 05's load-shape fixture class and the preemption-adjacent runbooks.
- **Input from `mod-107`.** The vendor dependency map, fallback contract, and provider-status ingestion are read by `rb-provider-degraded-fallback`, by the incident-command's vendor-communication rule, and by chapter 06's regulator-facing narrative.
- **Output to `mod-109`.** Chaos engineering uses this module's runbook library and fixture back-feed as the menu of failure modes to inject and the verification pattern to run against.
- **Output to `mod-110`.** The security-safety-compliance crossover takes chapter 06's handoff discipline deeper on the policy side.
- **Output to `mod-111`.** The release-engineering and progressive-delivery module hardens the gates whose absence this module's post-mortems repeatedly name as the failure to catch model-weight incidents earlier.
- **Output to `mod-113`.** DR / BCP for AI consumes the IC discipline and the peer-boundary map from this module.
- **Output to `mod-114`.** The cross-functional leadership module operates at the level this module's peer-boundary chapter sets.
