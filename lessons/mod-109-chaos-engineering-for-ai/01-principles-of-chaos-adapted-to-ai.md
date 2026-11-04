# Principles of Chaos Engineering, Adapted to AI Workloads

## Why this chapter exists

Chaos engineering is a mature discipline. The field has a public manifesto — the Principles of Chaos Engineering (principlesofchaos.org) — a canonical O'Reilly book (Rosenthal and Jones, *Chaos Engineering: System Resiliency in Practice*), a decade of case studies from Netflix / Amazon / Google / LinkedIn / Alibaba, and a CNCF project shelf (Chaos Mesh, LitmusChaos) that encodes the practices in Kubernetes-native CRDs. An AI SRE arriving at this topic does not get to re-derive the field. The job is to **take the five canonical principles as they are**, understand where their assumptions break when the workload is a model-backed service, and extend each with the AI-specific shape that the rest of this module builds on.

This chapter is the specification. Chapter 02 wires it into tooling; chapters 03–06 turn the principles into a catalogue of AI-specific experiments and a continuous program. The exercise for this chapter (exercise 01) is to write, for a chosen fleet, the extended principles document that the function will sign off on and that the chaos program will defend.

The scope is deliberate. These principles cover *reliability* behaviour — the pod that gets evicted, the provider that times out, the KV cache under pressure, the retrieval index that drops. Adversarial robustness (prompt-injection, model-extraction, jailbreak) and model-evaluation robustness (benchmark robustness, red-team coverage) are at the `ai-infra-security` and `ai-eval-engineer` peers respectively; chapter 06 draws those boundaries explicitly.

## 1. The five canonical principles

The Principles of Chaos Engineering, in their published form, read as five commitments. They are summarised here in order; the authoritative text is at principlesofchaos.org and the deeper rationale is in the Rosenthal / Jones book. An AI SRE should treat the published principles as the source of truth and this chapter as a translation layer.

1. **Build a hypothesis around steady-state behaviour.** Before you break anything, define the system's steady state as a measurable output — the user-visible metric that tells you the system is working. Formulate your hypothesis in terms of that metric: "under failure X, the steady-state metric remains within tolerance."
2. **Vary real-world events.** Inject faults that correspond to the failure modes the system will actually see in production — hardware faults, network degradation, dependency timeouts, partial outages, resource exhaustion — not contrived or synthetic errors.
3. **Run experiments in production.** The target environment is the only environment that has production traffic, production data, production scheduling, and production configuration. Staging is a sketch; production is the painting.
4. **Automate experiments to run continuously.** Manual game-days are a necessary start but not the end state. A chaos program that only runs when a human remembers it is a program that regresses quietly between the times it runs.
5. **Minimise the blast radius.** The experiment should give you maximal evidence per unit of customer pain. Start small — one pod, one route, one shadow request — and grow the scope only as the hypothesis is validated at each tier.

These five hold up for classical SRE work almost unchanged. They hold up for AI work with specific extensions, which the rest of this chapter writes.

## 2. Principle 1 — Steady state, adapted for non-deterministic systems

Classical steady state is often a single number: request-success-rate, p95 latency, order-conversion rate. The system is deterministic enough that the same input produces the same output, and small variance in the metric is attributable to transient noise. A chaos experiment moves the metric; you measure how far.

Model-backed services break that framing in two ways:

- **The output is sampled, not deterministic.** The same prompt produces a distribution of outputs even with temperature zero in some provider configurations (batch-size effects, non-deterministic kernels, FP8 or BF16 reduction orders). "Does the answer match?" is the wrong hypothesis.
- **Quality is a signal, not just availability or latency.** A system that answers every request with a 200 in 500 ms but hallucinates 10% more during a chaos experiment has failed steady state in a way that the classical SRE's availability dashboard does not surface.

The AI-adapted steady state is therefore **three parallel metric families** rather than one:

1. **Serving-health steady state.** Request success-rate, TTFT p95, TPOT p95, streaming-stall rate, prompt-cache hit rate, structured-output conformance rate. These are the signals from `mod-104` that answer "is the service up and fast."
2. **Quality steady state.** Judge-pipeline scores (groundedness, faithfulness, helpfulness), refusal-rate, safety-classifier output rate, task-success-rate on canary sets. These are the signals from `mod-105` chapter 07 and `mod-108` chapter 01's `rb-hallucination-rate-spike` / `rb-refusal-rate-spike` / `rb-toxic-output-blast` detection sections.
3. **Economic steady state.** Per-request cost, per-session cost, provider-spend-per-hour, vendor-quota consumption. An experiment that halves availability for ten minutes but triples cost per request for the next hour (because the fallback is more expensive) has a result the business cares about, and chaos engineering must surface it.

A good AI chaos hypothesis names metrics from each family. "Under primary-provider timeout injection on route X, (a) request success-rate stays above the 1-h SLO burn threshold, (b) judge-pipeline groundedness stays within one standard deviation of baseline, and (c) per-request cost stays within 1.5× baseline." All three or the hypothesis is incomplete.

### 2.1 Baselines are harder to compute for quality

Availability and latency baselines come from `mod-104`'s time-series store on a trailing 7-day window and are cheap. Quality baselines require a judge pipeline running on live traffic, a sampled canary eval set, or both. The chaos program must inherit the baseline from `mod-105`'s offline-eval harness and `mod-108` chapter 05's fixture ledger; do not synthesise a bespoke quality baseline inside a chaos tool. Two baselines diverge; one baseline binds.

### 2.2 Non-deterministic outputs break naive deltas

A "did the answer change?" comparison fires on every retry even without an experiment. The AI chaos program uses **distributional comparisons** — judge-score distribution, structured-output-validity rate, refusal-rate, cost distribution — over a window large enough to be statistically meaningful. "The p95 judge score shifted by two standard deviations" is a chaos result; "this one sample is different" is noise.

### 2.3 The streaming case

Streaming responses have their own steady state. "Stream closed with a terminal event" and "inter-token pause did not exceed N × baseline" are two metrics the chaos program must track that a request/response service does not need. See `mod-108` chapter 01's `rb-streaming-stall` for the signal set.

## 3. Principle 2 — Vary real-world events, AI edition

Classical chaos injects hardware fault, network fault, dependency fault, and resource-exhaustion fault. Each maps cleanly to a Chaos Mesh / Litmus CRD (`PodChaos`, `NetworkChaos`, `IOChaos`, `DNSChaos`, `StressChaos`, `TimeChaos`). An AI fleet shares this surface — the GPUs run on nodes, the services talk over a network, the storage has an fsync path — but it adds a parallel surface no classical catalogue covers:

- **GPU-specific hardware events.** XID errors (benign and fatal), ECC double-bit errors, NVLink errors, driver watchdog triggers, MIG partition misconfigurations, DCGM counter anomalies. NVIDIA's XID documentation is the authoritative taxonomy; the AI SRE chaos program injects a representative subset.
- **Preemption and scheduler events under GPU pressure.** A higher-priority training job starting on the same cluster, a spot reclamation on serving nodes, a PDB that was set too loose, a priority class that was set too high on a batch workload.
- **Provider-fallback events.** Primary provider timeout, primary provider rate-limit (429), primary provider overloaded (5xx), primary provider version-deprecation, primary provider silent-quality regression. The chaos program injects each class and validates the fallback contract from `mod-107` chapter 02.
- **Cache events.** Prompt-cache invalidation, prompt-cache poisoning via a synthetic template perturbation, KV-cache memory pressure under long context, cache-replica desync.
- **Retrieval events.** Vector-index drop, vector-index replica failure, embedding-service timeout, ingest pipeline stall, synthetic staleness (ingest paused for N hours), chunking-change simulation.
- **Agent events.** Tool-call error-shape injection, tool rate-limit injection, external-API delay, recursive-tool detector verification, agent-session cost-cap enforcement test.
- **Eval-pipeline events.** Judge-model unavailable, judge-model drift simulation (score offset), eval-set checksum mismatch, calibration-set drift.

The AI SRE's chaos catalogue is the union of the classical catalogue and this parallel set. Chapter 03 enumerates it; each entry in that catalogue traces back to a failure class named in `mod-105` or an operational surface in `mod-103`, `mod-104`, `mod-106`, or `mod-107`. **An experiment that injects a fault no production system has ever seen is not grounded chaos — it is theatre.**

### 3.1 The "real" bar

The "real-world events" principle is strict for a reason: contrived faults teach the system to survive contrivances and leave the real failure modes uncovered. The AI chaos program defends the "real" bar with two rules:

- **Every experiment cites a source.** A post-mortem (`mod-108` chapter 03), a public post-mortem (`mod-108` chapter 04), a vendor status-page history item (`mod-107` chapter 04), a dashboard panel that has fired before, or a near-miss reported in the on-call channel. If no source exists, the experiment is deferred until it does or is justified under a "known-unknown" note in the catalogue.
- **The fault shape matches production.** A provider timeout in production is TCP-level — the connection stays open while the HTTP body stalls; a naive "sleep 30s then 500" misses the half-complete-stream failure mode. The chaos tooling (chapter 02) is chosen to reproduce the production shape, not a shape convenient to inject.

## 4. Principle 3 — Production first, with AI-specific blast-radius caveats

The canonical principle is uncompromising: production is the only environment that reproduces the production system. Staging is a sketch. The AI SRE's extension is that **some AI faults have user-visible safety implications that classical faults do not**, and the blast-radius discipline must therefore be tighter, not that the principle flips.

The two gradations the AI SRE chaos program uses:

- **Tier-1 production-safe experiments.** Experiments whose worst case is bounded by the product's existing degraded-mode envelope: a pod gets evicted, a provider gets timed out, a cache gets flushed, a replica gets drained. The user-visible effect is latency or a temporary quality dip, both of which the service's SLOs already budget for. Run these in production under blast-radius controls.
- **Tier-2 staging-only experiments.** Experiments whose worst case is a user-visible safety event: a safety-filter disable, a toxic-output canary, a prompt-cache poisoning that could leak PII, a corrupted-weight load. These run in a staging environment that mirrors production topology but does not serve real users, or in a shadow pod that receives duplicated but discarded traffic. The canonical "production first" bias is honoured by making the staging environment itself as production-fidelity as the team can afford.

The decision rubric for the tier is written down in the chaos-program policy document (chapter 05 §5) and is signed by the AI SRE function lead and the safety / policy peer (`mod-108` chapter 07 §1.12). An experiment moving from tier-2 to tier-1 is a reviewed change, not a unilateral one.

### 4.1 The darker-blast-radius controls

Beyond the tier rubric, each tier-1 experiment runs with four hard limits:

- **Cohort cap.** Which tenants, which routes, which regions, which account classes the experiment can affect. Internal / dog-food cohorts first.
- **Traffic-fraction cap.** A percentage of live traffic (e.g., 1%) that can be subjected to the experiment at once.
- **Time cap.** The experiment's maximum duration (e.g., 15 minutes) before an automatic halt.
- **Auto-halt condition.** The SLO-burn or safety-signal threshold that terminates the experiment immediately. Chapter 05 §3 writes the halt-controller shape.

A chaos platform that cannot enforce these limits mechanically is not a chaos platform; it is a failure-injection script. Tool choice is subordinate to this.

## 5. Principle 4 — Automate continuously, AI-specific cadence

A chaos program that only runs when the SRE lead schedules a game-day degrades between game-days as the underlying system changes. The automation principle says the experiments must run on a cadence that is at least as fast as the system's change cadence. For AI services, the change cadence is *faster* than most classical services because:

- **Model versions change weekly or faster.** Provider auto-updates (`mod-107` chapter 06 §3) and internal model deploys mean the serving substrate behind a given route moves on a shorter horizon than a traditional application image.
- **Prompt templates and agent scaffolding change daily.** `AI Engineering` lands dozens of template commits a week on an active product. Every change is a potential cache invalidation or refusal-rate shift (`mod-108` chapter 01 `rb-cache-hit-cliff`, `rb-refusal-rate-spike`).
- **Retrieval indices refresh continuously.** Ingest and re-index run on a short cycle; each run is a potential source of the staleness or embedding-drift classes (`mod-108` chapter 01 `rb-retrieval-index-staleness`, `rb-embedding-drift`).

The chaos program's cadence must match. The AI SRE's default:

- **Continuous (per-change).** A small subset of experiments — provider-timeout, pod-kill, prompt-cache-flush — runs on every serving-image deploy in a canary stage before promotion.
- **Daily.** A rotating subset of the full catalogue runs against the pre-production cluster overnight, with results posted to the on-call channel.
- **Weekly.** A larger subset runs in production during a low-traffic window, with the blast-radius controls from §4.
- **Quarterly.** A full game-day with a chosen scenario (chapter 05 §4), SME participation, and a published retrospective.

The cadence is enforced by scheduling (Chaos Mesh's `Schedule` CRD, Litmus's `ChaosSchedule`, AWS FIS's scheduled templates, or a plain cron trigger) and surfaced in the function's quarterly reliability review. Chapter 05 writes the governance shape; this principle sets the floor.

### 5.1 The "flaky chaos" failure mode

Automated chaos that flakes — passes most of the time, fails on an occasional unrelated signal — teaches the team to ignore it. The chaos program's investment in automation is matched by an equal investment in stable baselines (§2) and tight auto-halt signals (§4). An experiment that cannot be run three times in a row and reach the same verdict is not ready for automation; it stays in manual rotation until it is.

## 6. Principle 5 — Minimise blast radius, under AI-amplified consequences

The blast-radius principle is a *technique*, not a limit. Chaos is not safer because you did less of it; it is safer because you got the evidence you needed with the least customer impact. The AI SRE's extension is that the consequences of over-shooting the blast radius are **wider** on AI services than on classical ones, because:

- **Model-weight-level corruption cannot be rolled back the way a container image can.** A deploy that pushes a bad weight and takes ten minutes to roll back has served ten minutes of user-visible wrong answers that the product may not be able to retract.
- **Agent side-effects can be irreversible.** An agent that calls `send_email`, `pay_invoice`, `publish_post`, `delete_record` has moved a part of the world that chaos engineering cannot un-move.
- **Safety-adjacent regressions are a legal and PR event, not just a reliability one.** `mod-108` chapter 06 writes the handoff; the chaos program must not create the conditions that trigger it.

The AI-adapted blast-radius controls therefore layer on top of the classical ones:

- **Classical layer.** Single-pod / single-node / single-zone, 1% traffic cap, time-boxed, automatic halt on SLO burn.
- **AI-specific layer.** Internal-cohort-first, no-side-effect-tools-during-experiment, safety-filter-locked-strict during tier-1 runs, auto-halt on safety-classifier spike (not only on SLO burn).

The order of operations at every experiment is the same:

1. **Smallest scope that produces a signal.** A single pod kill before a zone drain. A single provider route timeout before a cross-provider outage. A single tool error before an all-tools error.
2. **Signals in both directions.** Does the experiment have a proof-of-containment metric (nothing else moved) and a proof-of-effect metric (the thing we were trying to break did break)? An experiment with only one of the two is uninterpretable.
3. **Escalation only after confirmation.** Grow the scope by one tier (one node → one zone, 1% → 10%, one route → one product area) only after each step's hypothesis holds.

### 6.1 The "no-chaos list"

Not every surface is a chaos target. Some surfaces (customer accounting ledgers, moderation-queue integrity, model-registry write paths, payment-authorisation flows that are reached through an agent tool) are listed in the chaos-program policy as **no-chaos targets** — they are protected by other disciplines (strict testing, audit, write-path verification) and are not subject to failure injection. The list is maintained alongside the catalogue; chapter 03 §8 names the AI-specific entries that every program starts with.

## 7. The sixth, AI-only principle: "learnings back to the runbook"

The canonical five principles stop at "automate continuously." For the AI SRE, a sixth principle is load-bearing because the function's whole value proposition depends on it:

- **Every chaos finding produces at least one permanent fixture.** An alert, a canary case, an eval-set entry, a runbook amendment, a chaos-schedule change itself. The fixture back-feed loop (defined in `mod-108` chapter 05) is the mechanism; the chaos program is the generator.

Without this principle, chaos is a hobby. With it, chaos is the production arm of the SLO stack. The learnings-to-runbook loop is detailed in chapter 05; naming it as a principle here commits the program to run it.

## 8. The AI SRE's principle-extension worksheet

When the function adopts these principles for a specific fleet, the worksheet looks like this. Exercise 01 produces the filled version.

1. **Steady state.** Name the three metric families (§2). Pin the serving-health signals to a `mod-104` artifact, the quality signals to a `mod-105` canary / judge pipeline, and the economic signals to a `mod-107` cost source. Confirm each has a 7-day baseline in the time-series store.
2. **Real-world events.** List the ten failure classes most recently observed in the on-call channel, a post-mortem, or a vendor status page. Classify each as classical (Chaos Mesh / Litmus / FIS covers it), AI-specific (requires the AI catalogue in chapter 03), or out-of-scope (adversarial / eval-robustness, §6 of chapter 06).
3. **Production-first tier.** Decide which experiments run in production with blast-radius controls (tier-1) and which must stay in staging (tier-2). Get the safety peer's signature.
4. **Automation cadence.** Choose the per-change / daily / weekly / quarterly split. Name the scheduling mechanism and the result-publication channel.
5. **Blast-radius policy.** Write the cohort, traffic-fraction, time, and auto-halt caps per tier. Write the no-chaos list.
6. **Back-feed commitment.** Confirm that every experiment's result goes to the fixture ledger (`mod-108` chapter 05 §3) and that the ledger is reviewed quarterly.

A chaos program without this worksheet is a tool deployment. With it, it is a reliability practice.

## Summary

- The five canonical principles (steady state, real-world events, production first, automate, minimise blast radius) hold for AI workloads; each needs an AI-specific extension rather than a replacement.
- Steady state on AI services is three metric families — serving, quality, cost — not one. A hypothesis that names only latency is incomplete.
- Real-world events on AI services add a parallel catalogue to the classical fault set: GPU events, preemption, provider fallback, caches, retrieval, agents, evals. Chapter 03 enumerates it.
- Production first stays the rule, but tier-2 experiments (safety-adjacent, side-effect-creating) stay in staging under a documented policy.
- Automation cadence must match the system's change cadence, which for AI services is faster than the classical SRE's default.
- Blast-radius controls layer AI-specific caps (internal-cohort first, side-effect-disabled, safety-filter-strict, safety-signal auto-halt) on top of classical caps.
- A sixth AI-only principle — learnings back to the runbook — closes the loop with `mod-108` chapter 05's fixture ledger.

Chapter 02 picks the tooling (Chaos Mesh, LitmusChaos, Gremlin, AWS Fault Injection Service) that lets a Kubernetes-hosted AI fleet execute these principles mechanically.
