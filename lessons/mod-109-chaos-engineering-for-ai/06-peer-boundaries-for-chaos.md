# Peer Boundaries for AI Chaos Engineering

## Why this chapter exists

A chaos program is tempting to grow past its charter. The tools are generic; the skills are transferable; the dashboards are shared with nearby teams. If the function's chaos lead can run a provider-timeout experiment, surely they can also run a prompt-injection experiment? And if they can run a prompt-injection experiment, can they run a jailbreak suite? And if they can run that, why not add benchmark-robustness scans?

The answer is no, and the reasons matter. Adversarial robustness is a *different discipline* with different principles, different tooling, different review cadences, and different outcome measures. Model-evaluation robustness is a third discipline again. A chaos program that drifts into either is a program that is doing two things badly instead of one well.

This chapter closes the module by drawing the line. It names the two peer disciplines whose work shares the surface of "injecting faults into an AI system to learn something," describes what each owns and what the AI SRE's chaos program *does not* own, and writes the small number of shared artifacts where the peers and the AI SRE must collaborate.

The chapter mirrors `mod-107` chapter 07 and `mod-108` chapter 07 in function and tone: vocabulary first, handoff artifacts second, anti-patterns last.

## 1. The three disciplines on the same substrate

Three programs can inject faults, delays, and perturbations into a model-backed service. They look similar from the outside; they differ sharply in intent, method, and ownership.

### 1.1 AI SRE chaos (this module)

- **Intent.** Verify that the reliability behaviour the function owns — fallbacks, breakers, scheduler decisions, degraded-mode routing, cache behaviour, retrieval degradation — holds under the failure modes production will actually see.
- **Method.** Infrastructure and dependency fault injection (chapter 02 §1), composed into AI-specific experiments (chapter 03), run on a cadence (chapter 05), measured in SLO-grounded metrics (serving, quality, cost).
- **Owner.** The AI SRE function.
- **Outcome measure.** MTTD, MTTR, fallback-hit-rate, fixture back-feed rate, runbook coverage (`mod-108` chapter 05's ledger).

### 1.2 Adversarial robustness testing (`ai-infra-security` peer)

- **Intent.** Verify that the system holds against *actors* — prompt injection, model extraction, membership inference, jailbreak, data exfiltration via agents, prompt-leakage attacks, supply-chain tampering against artifacts. These are attacks, not accidents.
- **Method.** Red-team exercises, automated attack suites (Garak-style scanners, OWASP LLM Top 10 scanners, private attack catalogues), threat-model-driven test cases, bounty programs. The relevant external frameworks are OWASP's Top 10 for LLM Applications, NIST AI 100-2 (Adversarial Machine Learning), and MITRE ATLAS.
- **Owner.** The `ai-infra-security` peer (naming per `mod-101` chapter 02).
- **Outcome measure.** Attack-coverage, mean time to detect an attack pattern, time-to-remediate a disclosed finding, red-team-to-production-fix cadence.

The chaos program *touches the same tooling* in some places — a chaos operator can inject an HTTP fault that looks like an attack — but the intent and the outcome measure differ. An adversarial-robustness test that passes is a system that resisted an actor; a chaos test that passes is a system that survived a failure. Pretending these are the same leaves the function with two half-built programs.

### 1.3 Model-evaluation robustness (`ai-eval-engineer` / `model-evaluation-engineer` peers)

- **Intent.** Verify that the *model* behaves as expected across a representative distribution of inputs and specifically under edge-case and distribution-shift conditions. This is the robustness of the model's outputs, not the robustness of the system that serves them.
- **Method.** Benchmark sweeps, canary eval sets with perturbation coverage, held-out evaluation, red-team eval harnesses from safety, counterfactual-sample generation, drift detection.
- **Owner.** `ai-eval-engineer` and / or `model-evaluation-engineer` (naming per `mod-101` chapter 02).
- **Outcome measure.** Eval-score distributions, benchmark-robustness scores, drift metrics, coverage against a documented eval taxonomy.

A model-evaluation robustness run tests whether `the-model-under-test` is good enough. A chaos run tests whether `the-system-serving-the-model` is good enough. Again: same tools can appear on both surfaces; different intent, different ownership.

## 2. What the AI SRE chaos program does *not* do

A concrete list is more useful than a principle. The AI SRE chaos program does not:

- **Inject prompt-injection payloads.** Not even to "see what happens." The `ai-infra-security` peer owns the payload catalogue, the test harness, and the remediation path.
- **Run jailbreak suites.** Even in staging. Jailbreak authors need a disciplined red-team process; a casual chaos harness that produces jailbreak attempts creates data-handling and PR exposure the function should not create.
- **Test model-extraction resistance.** Probing-style attacks on a model's weights or training data belong to security.
- **Run benchmark sweeps against the model.** Eval owns benchmarks. A chaos run that inadvertently measures "what does the model do on the MMLU subset" is drifting.
- **Author adversarial canary queries.** Canary eval sets with adversarial perturbations are a joint artifact (chapter 05 §1 of `mod-108` chapter 07 §1.6); the chaos program may *consume* them but does not *produce* them.
- **Run safety-evaluation drills.** The safety / policy peer owns the safety taxonomy and the red-team cadence against it.
- **Run model-extraction / membership-inference tests.** `ai-infra-security` owns these.
- **Grade model quality.** Ever. The judge-pipeline the chaos program depends on is owned by `ai-eval-engineer`.

A chaos run that touches any of these surfaces unintentionally is a chaos run that has outgrown its charter. The policy document from chapter 05 §3 lists the above as out-of-scope so the program does not drift.

## 3. The shared surfaces (where the peers meet)

Three surfaces are jointly owned. On each, the AI SRE chaos program is a consumer, a producer, or both.

### 3.1 Canary eval sets

- **Who owns.** `ai-eval-engineer`.
- **Chaos program's role.** Consumer. The chaos catalogue's quality hypothesis (chapter 03 §1) reads from the canary eval set's thresholds. When a chaos run degrades quality on the canary, the result is a finding for both the chaos program and the eval team; the shared artifact is the canary set itself.
- **Boundary clarification.** The chaos program does not *add* items to the canary set unless the eval team reviews and accepts them. `mod-108` chapter 05 §3's Candidate → Owned → Implemented cycle covers the review.

### 3.2 Safety classifier output as a halt signal

- **Who owns.** Safety / policy peer and `ai-infra-security` (depending on the organisation's split).
- **Chaos program's role.** Consumer. Chapter 05 §2 wires the safety-classifier spike as an auto-halt signal. The classifier itself and its thresholds are not the chaos program's.
- **Boundary clarification.** The chaos program does not tune the classifier. If a chaos run routinely triggers a safety-classifier spike without producing user-visible content, that is a classifier-tuning question, not a chaos question — route it to the safety peer.

### 3.3 Fault-injection tooling

- **Who owns.** The chaos operator (chaos-mesh / litmus / gremlin / FIS) is often shared infrastructure with the security team.
- **Chaos program's role.** Co-owner with `ai-infra-security` of the operator's install, RBAC, and audit configuration.
- **Boundary clarification.** Experiments are scoped per discipline. The chaos program's ServiceAccount cannot author adversarial-robustness experiments; the security team's cannot author reliability chaos. RBAC enforces the split so neither drifts.

### 3.4 Game-day scenarios that span disciplines

- **Who owns.** Composed scenarios (e.g., "provider-degraded + jailbreak attempt during degraded mode") are jointly owned. The AI SRE function's chaos lead, the security peer's red-team lead, and the safety / policy peer co-author.
- **Chaos program's role.** Reliability-leg owner. The security peer owns the adversarial leg; the safety / policy peer owns the safety leg.
- **Boundary clarification.** Joint game-days produce a joint retrospective; fixtures land in each peer's ledger per the finding's owner.

## 4. Anti-patterns at the boundary

The failure modes the AI SRE must recognise and prevent:

- **Chaos program running a red-team drill.** The chaos lead reads about an LLM jailbreak pattern on Twitter and runs it as a "chaos experiment." The result has no owner, no remediation path, no threat-model context, and — in regulated jurisdictions — may be a reportable compliance event.
- **Security's red-team running a reliability drill.** The mirror: security runs a provider-timeout drill because "it's all fault injection." The result lands outside the fixture ledger, the SLO stack never learns, and the fallback contract continues to be unverified.
- **Shared tooling with shared ServiceAccounts.** A single `chaos-admin` ServiceAccount used by both functions. One misconfiguration on one side's policy hits both; the audit log cannot tell them apart.
- **Shared dashboards with shared vocabulary.** The chaos program adopts the security team's "attack surface" language, the security program adopts "fallback-hit-rate" — each peer's metrics drift because the words no longer mean the same thing to both teams.
- **The chaos program consuming eval benchmarks as if they were canary evals.** The chaos program should not read from a public benchmark (MMLU, HELM, etc.) to form its hypothesis. The benchmark is for `ai-eval-engineer`; the chaos program reads the *function's canary eval*, which is grounded on the production workload.

## 5. When the peer lines must still be read backwards

Boundary chapters sometimes over-fence. Three cases where the boundary bends:

- **A chaos experiment's result implicates a safety-classifier gap.** The experiment finds that during degraded routing, the safety classifier mis-labels at 2× baseline. This is a safety-peer finding; the chaos program reports it, the safety peer owns the fix. The chaos program's role is to *detect*, not to decide the taxonomy.
- **A chaos experiment's result implicates an adversarial-robustness gap.** Rare but real: a chaos experiment that stalls the provider also observes that an unmodified prompt in the failure window exfiltrates a system-prompt fragment through the fallback. The chaos program's role is to escalate to security and to add the detection (not the fix) to the function's dashboard.
- **A model-eval regression correlates with a chaos result.** The eval team sees a judge-score drift during weeks the chaos program ran a particular experiment. The co-ownership here is on root-cause; the eval team owns the judge-interpretation question, the chaos program owns whether the mechanism of the experiment was interfering with the signal.

In each of the three, the pattern is the same: the chaos program detects the cross-boundary signal, names the peer, hands off the artifact, and maintains the detection on the function's dashboard. The *fix* belongs to the peer.

## 6. The AI SRE's irreducible chaos surface

To restate what this module does own, now that the exclusions are explicit:

- **The reliability chaos catalogue** (chapter 03).
- **The provider-fallback contract chaos suite** (chapter 04).
- **The chaos program's policy, cadence, and auto-halt wiring** (chapter 05 §§1–3).
- **The quarterly game-day shape** (chapter 05 §4).
- **The fixture back-feed loop with runbook library and SLO stack** (`mod-108` chapters 01 and 05).
- **The chaos program's quantitative narrative** (MTTD / MTTR / fallback-hit-rate trends from chapter 04; auto-halt and fixture-back-feed trends from chapter 05) to the function's leadership on the quarterly cadence.

Nothing above is co-owned. Everything above is accountable to the function.

## 7. The peer-boundary worksheet

Use this before launching the chaos program on a fleet, and re-read it when the program crosses a tier boundary (daily → weekly, weekly → production, pre-production → quarterly game-day):

1. **Does this experiment inject an actor's behaviour?** If yes, it belongs to `ai-infra-security`. Hand it over or add a co-authored scenario.
2. **Does this experiment measure model-output quality on an evaluation set?** If yes, read from `ai-eval-engineer`'s canary set; do not author one.
3. **Does this experiment touch the safety taxonomy or the safety-classifier policy?** If yes, co-run with the safety / policy peer; the chaos program consumes the classifier, it does not configure it.
4. **Does this experiment's result have a security or safety finding embedded in it?** If yes, escalate to the owning peer and add a detection to the dashboard; the chaos program does not own the fix.
5. **Is the tooling shared with other peers' programs?** If yes, RBAC and audit the ServiceAccount split before running. The chaos program's ServiceAccount cannot author other peers' experiments.
6. **Is this a joint game-day scenario?** If yes, co-author the runsheet, publish a joint retrospective, and route fixtures to the correct ledger.

Running this worksheet every time the program grows is the discipline that keeps the function's charter honest.

## Summary

- Three disciplines share the "inject faults into an AI system" surface: AI SRE chaos (this module), adversarial robustness (`ai-infra-security`), and model-evaluation robustness (`ai-eval-engineer` / `model-evaluation-engineer`). Each has a different intent, method, owner, and outcome measure.
- The AI SRE chaos program explicitly does not inject prompt-injection payloads, jailbreak attempts, model-extraction probes, benchmark sweeps, adversarial canary authoring, or safety-taxonomy tuning.
- Shared surfaces — canary eval sets, safety-classifier halt signals, fault-injection tooling, cross-discipline game-day scenarios — have jointly authored artifacts and separately owned fixes.
- Anti-patterns at the boundary (chaos running red-team drills, security running reliability drills, shared ServiceAccounts, shared vocabulary drifting) are named here so the function recognises them when they appear.
- When a chaos result implicates security, safety, or eval work, the chaos program detects and hands off; the peer owns the fix.
- The AI SRE's irreducible chaos surface is the catalogue, the contract-chaos suite, the program's policy and cadence, the game-day shape, the fixture back-feed loop, and the quantitative narrative to leadership.

This closes module 109. The chaos program it specifies is the practice that makes every other module in the AI SRE track verifiable in production, and the fixture back-feed it depends on makes the function's work compound over time rather than reset at every incident.
