# The Peer Deferral Map — Who You Hand It To

## Why this chapter exists

Chapter 01 gave you the sentence: *AI SRE owns the operational reliability contract of production AI systems.* The next practical problem is the one that lands in your Slack every week — an incoming concern that is *almost but not quite* yours. A model-runner is dropping requests. A kernel seems to have regressed under long prompts. The guardrail-sidecar is adding 80 ms to P99. The judge model's agreement with humans is drifting. The LLM is "hallucinating more" in a specific flow.

Each of these has an owner. In almost every case, the owner is not you — you are the *router*. This chapter lists every near-neighbour on the ladder, says what they own, says what you own at the seam, and gives the heuristic that resolves the ambiguous cases.

Keep in mind the **ownership rule** this org uses: assign primary coverage to the *lowest* level that genuinely requires the skill, and link rather than duplicate at higher levels. The same rule applies horizontally — the specialist peer owns their craft; you own the reliability contract around it.

## The level-35 peer trio (horizontal defers)

All three are the same level as you. They are specialists in their own craft; you are a specialist in reliability. Each of the three owns the inside of a single artifact; you own the operational envelope around all three.

### `ai-infra-inference-engineer` — serving-system authoring

| | |
|---|---|
| **Scope** | The serving engine itself at source-level depth — `vLLM`, `SGLang`, `TensorRT-LLM`, `Dynamo`, `Triton`. Scheduler loop, KV allocator, model-runner, worker-executor layering. |
| **What they hand you** | A deployable engine artifact with knobs: max batch, prefix-cache sizing, speculative-decoding config, tensor-parallel degree, chunk size, preemption policy. |
| **What you hand them** | Production-signal bug reports with repro conditions: a workload shape, a knob matrix, and the SLO that broke. |
| **The seam** | They decide *how the engine schedules*; you decide *which engine runs in production, with which knobs, at which capacity headroom*. |
| **Ambiguous-case heuristic** | If the fix is in C++ / Python inside the engine repo, it's theirs. If the fix is in the Helm chart, the HPA policy, the fleet mix, or the SLO definition, it's yours. |

### `ai-infra-performance` — kernel-authoring craft

| | |
|---|---|
| **Scope** | CUDA, CUTLASS, CuTe DSL, Triton kernels, Nsight-Compute to SASS. Authoring and tuning the kernels the engine calls. |
| **What they hand you** | A kernel that ships with a release-note — "we re-tuned this attention variant; prefill-TTFT improves by *X* on sequence length *Y*." |
| **What you hand them** | GPU telemetry showing a regression (DCGM SM activity, HBM pressure, per-kernel occupancy) and the workload slice where it shows up. |
| **The seam** | They decide *how fast one op runs on one GPU*; you decide *whether the fleet as a whole meets its SLO, at what cost, under what traffic mix*. |
| **Ambiguous-case heuristic** | If the regression lives inside a single kernel's correctness or occupancy, it's theirs. If the same kernel is fine in isolation and only misbehaves under production batch shapes, it's a joint ticket — you supply the trace, they supply the fix. |

### `ai-infra-security` — adversarial defence

| | |
|---|---|
| **Scope** | Prompt-injection classifiers at the API edge, PII-filter sidecars, jailbreak classifiers, model-extraction query-rate control, ML-BOM signing and supply-chain verification. |
| **What they hand you** | A guardrail with a documented latency budget, a cache-impact profile, and a fail-open vs. fail-closed policy. |
| **What you hand them** | The reliability *knock-on* — "your PII filter added 40 ms to P95 and dropped prefix-cache hit by *n*%; the SLO is now breaching; here is the trace." |
| **The seam** | They decide *what gets blocked*; you decide *whether the blocking is still inside the error budget*. Shared post-mortems when a safety incident and a reliability incident collide. |
| **Ambiguous-case heuristic** | A security control ticket is theirs. A latency / cache / cost ticket is yours. A joint incident (security fail-closed cascades into availability loss) is joint — you take IC, they take the security-side forensics. |

## The level-30 peer band (down-defers)

Lower on the ladder than you, these are peers from adjacent families you hand reliability-gate signals to or consume handoff contracts from. You do **not** own their craft, but you own the interface.

### `ai-infra-ml-platform` (peer, level 30, AI Infrastructure family) — training-serving handoff

| | |
|---|---|
| **Scope** | Self-service ML platform, model registry, experiment-tracking, training-cluster plumbing, promotion-to-serving workflow. |
| **What they hand you** | A model artifact with a resource profile (memory footprint, precision, throughput at reference concurrency) and a promotion manifest. |
| **What you hand them** | Reliability acceptance criteria for a promotion — the SLO it has to meet in shadow or canary before it flips to primary. |
| **The seam** | They own *how an artifact gets to production*; you own *whether production accepts it*. |
| **Ambiguous-case heuristic** | Artifact-handoff bug (wrong format, missing metadata, signed-by-the-wrong-key) is theirs. A shadow-deployment that passes offline eval but fails SLO in canary is yours to diagnose, theirs to fix if the fix is upstream of serving. |

### `ai-eval-engineer` (peer, level 30, AI Engineering family) — application eval release-gate

| | |
|---|---|
| **Scope** | Application-layer eval sets, judge-model prompting, prompt-regression harnesses, the eval-set lifecycle itself. |
| **What they hand you** | An eval pass/fail signal with a confidence band, delivered as structured output you can wire into a release gate. |
| **What you hand them** | An SLO-shaped integration contract — "a release candidate needs a passing eval signal this fresh, with this latency, to be allowed to roll past 5% canary." |
| **The seam** | They decide *what "the output is good" means*; you decide *whether a drop in that signal trips the error budget and auto-rolls back*. |
| **Ambiguous-case heuristic** | The eval itself drifting (judge-model agreement with humans decays, eval set rots) is theirs to fix. The release-gate wiring to the eval signal is yours. |

### `model-evaluation-engineer` (peer, level 30, ML Engineering family) — benchmark / statistical / judge-vs-human methodology

| | |
|---|---|
| **Scope** | Benchmark construction, statistical calibration of eval signals, judge-vs-human agreement methodology. |
| **What they hand you** | A statistically-defensible eval methodology. |
| **What you hand them** | Production telemetry that can be joined back to their methodology (per-request eval tags, drift signals). |
| **Ambiguous-case heuristic** | Methodology questions are theirs. Operational wiring of the signal is yours. |

## The level-25 peer (down-defer) — CI/CD automation

### `ai-infra-mlops` (peer, level 25)

| | |
|---|---|
| **Scope** | CI/CD for ML — the pipeline that trains, registers, promotes, and deploys model artifacts, the dataset / feature-store plumbing underneath it. |
| **What they hand you** | A pipeline you can read, modify, and wire reliability gates into. |
| **What you hand them** | The *gate* — the SLO-based acceptance check, the eval-signal check, the canary / shadow / blue-green orchestration config — that their pipeline has to call out to. |
| **The seam** | They own the *pipeline*; you own the *gates in the pipeline*. |
| **Ambiguous-case heuristic** | Pipeline authoring is theirs. Gate authoring is yours. If a gate is wrong, you fix it; if the pipeline is wrong, they fix it. |

## The AI Engineering consumer tracks (level 25) — your users

### `llm-application-developer`, `rag-engineer`, `applied-ai-engineer`

These are the *consumers* of your platform. They are not peers in the "you share the stack" sense; they are the customers your SLOs are for.

| | |
|---|---|
| **Scope** | Prompt engineering, agent design, tool-call orchestration, RAG wiring, product-side observability. |
| **What they hand you** | A traffic shape — prompt-length distribution, tool-call branching factor, expected cache-hit rate, QPS curve. |
| **What you hand them** | A platform with documented SLOs, a status-page ingest, a runbook for the incident classes they are likely to notice first (silent output drift, provider degraded, latency regression). |
| **The seam** | They own *what the model is asked to do and what the output looks like downstream*; you own *whether the ask gets delivered reliably, fast, and in budget*. |
| **Ambiguous-case heuristic** | "The model is giving bad answers in flow X" starts as theirs — the prompt or the retrieval is usually upstream of you. It becomes yours if the signal is clearly platform-wide (every flow regresses at the same minute) rather than flow-specific. Chapter 05's charter explicitly names this routing rule. |

## The seams that are not peers

Three seams that are neither peers nor consumers but still land in your queue:

- **Platform finance / FinOps.** They own the cost model; you own the cost SLI and the levers (fleet mix, quota, batch policy) that move it. You hand them cost-per-1M-tokens and cost-per-successful-request; they hand you the budget envelope.
- **Legal / compliance / privacy.** They own data-residency rules, retention, regulator sign-off. You hand them telemetry that *proves* the reliability contract is being met in a residency-respecting way; you do not interpret statute.
- **SecOps / DFIR.** They own security-incident evidence as the system of record. You contribute AI-specific telemetry to their investigations; you are not the evidence-of-record.

## The routing rule, compressed

When a reliability-adjacent concern lands in your queue, ask the three questions in order:

1. **Is this operational?** If it is a design-time / training-time / product-decision question, it is not SRE — route to the appropriate design, ML, or product owner.
2. **Does the fix live inside one of the peer crafts** (engine source, kernel, security control, eval methodology, pipeline, prompt, model artifact)? If yes — route to that peer, with a reproducible signal attached.
3. **If the answer to both above is no, it is yours** — SLO authoring, incident response, capacity, chaos, release gate, DR, program ritual.

This is the three-line version of the artifact you will write in `exercise-03`.

## Visual: the routing diagram

```
                  Incoming reliability concern
                              │
                              ▼
            ┌──────────── operational? ─────────────┐
            │                                       │
            no                                     yes
            │                                       │
            ▼                                       ▼
        route to design /        ┌────── fix inside a peer craft? ──────┐
        training / product       │                                       │
                                yes                                     no
                                 │                                       │
                     ┌───────────┼───────────┬───────────┬──────────┐    ▼
                     ▼           ▼           ▼           ▼          ▼  YOU
             inference-engineer  perf       security    ml-platform eval-engineer /
             (engine source)   (kernel)    (control)   (artifact)   mlops / app
```

## Summary

- Three level-35 peers (`inference-engineer`, `performance`, `security`) each own a *craft*; you own the reliability contract around all three.
- Three level-30 peers (`ml-platform`, `ai-eval-engineer`, `model-evaluation-engineer`) each own a *handoff artifact*; you own the interface that accepts or rejects it.
- `ai-infra-mlops` at level 25 owns the *pipeline*; you own the *gates in the pipeline*.
- The AI Engineering consumer tracks at level 25 are your *users*, not your peers — they own what the model is asked to do; you own the delivery contract.
- The three-question routing rule (operational? peer craft? else yours) is the compressed form of the charter you write in `exercise-03`.
