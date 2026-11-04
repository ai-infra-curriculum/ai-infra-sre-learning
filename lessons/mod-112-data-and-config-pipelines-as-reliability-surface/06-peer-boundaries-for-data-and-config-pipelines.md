# Peer Boundaries for Data and Config Pipelines

## Why this chapter exists

The module has written four reliability surfaces — prompts and configs, retrieval, feature and metadata stores, eval sets — that the AI SRE function owns the contract on. On each surface, the authoring is done by a different peer discipline. If the function is not disciplined about the boundary, four things go wrong at once: the function drifts into authoring work it is not expert in, the peers drift into reliability work they are not accountable for, the audit trail loses its owner, and the single pane (chapter 05) fills with signals nobody is responsible for triaging.

This chapter writes the boundary explicitly, in the same shape as `mod-111` chapter 08. It names the peer disciplines that compose with this module, describes what each owns and what the function *does not* own, enumerates the shared surfaces, and lists the anti-patterns the function must recognise. It closes the module in the same way `mod-111`'s chapter 08 closes its module: boundary before capability, so the capability the function builds is defensible against the peers whose expertise it depends on.

## 1. The four peers at the pipeline surface

Four peer disciplines sit next to the function on the data and config surfaces. Each is a distinct role on the engineering ladder (`mod-101`); each has a specific handoff.

### 1.1 `rag-engineer` peer (level 25, consumer role per `mod-101`) <!-- needs-research: confirm the exact level and slug of the rag-engineer peer in the current ladder -->

- **What they own.** The retrieval pipeline's internals — the chunker, the embedding-model selection, the index-type decision (HNSW / IVF / flat / hybrid BM25+vector), the re-ranker choice and tuning, the retrieval probe-set authoring, the context-assembly strategy, the retrieval-prompt stitching.
- **Representative tooling.** FAISS, Milvus, Weaviate, Qdrant, Pinecone, pgvector, Elasticsearch; LangChain / LlamaIndex retrieval graphs; Cohere / BGE / Jina rerankers; the hybrid-search layers.
- **Handoff from this module.** Chapter 02's retrieval-freshness and retrieval-recall SLOs; chapter 02's embedding-index-version pin as config; chapter 02's rerank-latency-in-TTFT accounting.
- **Handoff to this module.** The function reads the retrieval pipeline's health via the signal set from chapter 02 §2; the function does not rewrite the pipeline.

### 1.2 `ai-infra-ml-platform` peer (level 30 per `mod-101`) <!-- needs-research: confirm level and slug -->

- **What they own.** The feature store itself — feature authoring, feature-pipeline code (Flink / Beam / Spark / Airflow / dbt jobs), the feature-registry content, the online-store and offline-store operation, the materialisation scheduling, backfills, feature-set versioning. The metadata stores (model registry, feature registry, lineage store) and the training / serving parity conventions.
- **Representative tooling.** Feast, Tecton, Vertex AI Feature Store, SageMaker Feature Store, Databricks Feature Store; Redis / DynamoDB / Bigtable / Cassandra as online-store substrates; parquet / BigQuery / Snowflake as offline-store substrates; MLflow / Vertex AI / SageMaker as registry substrates.
- **Handoff from this module.** Chapter 03's feature-serving SLO set (availability, freshness, completeness, parity); chapter 03's parity audit; chapter 03's pipeline-stall detector; chapter 03's metadata-store SLOs.
- **Handoff to this module.** The function reads the feature surface's health via the signal set from chapter 03; the function does not author features or operate the pipelines.

### 1.3 `ai-eval-engineer` peer (level 30 per `mod-101`) <!-- needs-research: confirm level and slug; same peer referenced in mod-111 chapter 08 -->

- **What they own.** Eval methodology — eval-set authoring, judge-pipeline design, judge-vs-human calibration, benchmark selection, the regression-set ledger, threshold-setting discipline for pass / fail, representativeness / discriminator / contamination metric choices.
- **Representative tooling.** Internal eval harnesses; OpenAI Evals, Inspect AI, Promptfoo, LangSmith evaluations, Weights & Biases Weave; judge-model stacks with human-calibration pipelines.
- **Handoff from this module.** Chapter 04's eval-set reliability signals (representativeness, discriminator, contamination); chapter 04's release-gate contract clauses; chapter 04's quarantine action on contamination detection; `mod-111` chapter 05's release-gate eval predicate.
- **Handoff to this module.** The eval peer does not operate the release pipeline or the on-call queue; they publish the eval-set content and the methodology signals.

### 1.4 `model-evaluation-engineer` peer (level 30 per `mod-101`) <!-- needs-research: confirm level and slug; mod-101 names both ai-eval-engineer and model-evaluation-engineer distinctly -->

- **What they own.** The methodology layer *below* eval-set construction — the public-benchmark review, the leaderboard methodology, the training / fine-tune evaluation harness, the model-selection evidence base. This is a sibling to `ai-eval-engineer`; where `ai-eval-engineer` sits closer to the application-level eval set, `model-evaluation-engineer` sits closer to the model-selection evidence base.
- **Representative tooling.** HELM, lm-evaluation-harness, HumanEval, SWE-bench, MT-Bench, Chatbot Arena, and the commercial equivalents; the model-evaluation framework internal to the fleet.
- **Handoff from this module.** Chapter 04's rot detection — when an eval set stops discriminating the active model population, the function routes to this peer with the discriminator / ceiling evidence so they can decide whether to harden the eval or retire it.
- **Handoff to this module.** The function reads their methodology outputs (the eval sets that enter the release-gate contract); the function does not run the benchmark selection.

### 1.5 Peers this module also touches, lightly

Three other peers have smaller but real handoffs:

- **`ai-infra-mlops` peer** (`mod-111` chapter 08 §1.2) — writes the CI/CD pipeline that ships prompt / config changes and feature-set updates. The function's chapter 01 progressive-delivery hook composes with their pipeline, the same way `mod-111` composes with their pipeline for model-weight changes.
- **`ai-infra-inference-engineer` peer** (`mod-111` chapter 08 §1.1) — the engine reads the (model, prompt, config) triple from this module's registry and ConfigMap; the engine's behaviour under a prompt / config change is the engineer peer's.
- **Application teams** — for each route, the application team owns the prompt content, the eval-set scope, and the feature-set selection. The function's contract is negotiated with the application team as the product-side counterparty.

## 2. What this module does *not* do

A concrete list is clearer than an abstract principle. The AI SRE's pipeline-reliability function explicitly does not:

- **Author prompts, exemplars, or system messages.** The content is the application team's, with eval-peer review. The function wires the registry, the release path, and the rollback; the function does not edit the string.
- **Author feature transformations, feature-store schemas, or the pipeline code.** The ML-platform peer owns these; the function reads the signals the pipeline emits.
- **Choose embedding models, chunk sizes, re-rank models, or index types.** The rag-engineer peer owns these; the function's chapter 02 enforces the versioning discipline on the choice.
- **Author eval samples, write judge prompts, or set judge-pipeline thresholds.** The eval peers own these; the function's chapter 04 enforces the reliability contract on the resulting eval sets.
- **Write the CI/CD pipeline that delivers prompt / config / feature-set / eval-set changes to production.** The mlops peer owns the pipeline; the function integrates with it through a well-defined release-gate interface.
- **Set retention policy on prompt-audit, config-audit, or eval-audit records.** `mod-110` chapter 05 owns retention; the function emits records against that discipline but does not set the retention target.
- **Operate the feature-store product itself (the Feast / Tecton / vendor control plane).** The ML-platform peer does; the function reads the serving-side signals that product exposes.
- **Decide the cadence of model training, feature backfills, eval refreshes, or retrieval-corpus curation.** These are product and peer decisions; the function responds to candidates and refreshes as they arrive.
- **Score eval candidates or interpret judge verdicts.** The eval peer does; the function's release-gate consumes the verdict as an input.

A pipeline-reliability program that drifts past any of these surfaces is a program whose scope the function cannot defend.

## 3. The shared surfaces

Five surfaces have shared ownership. On each, the function is a producer, a consumer, or both.

### 3.1 The prompt registry (chapter 01)

- **Who owns.** The function owns the registry as a service. The application team and the eval peer own the content.
- **Function's role.** Operate the service (availability SLO, latency SLO, audit emission, rollback runbook). Enforce the content-addressed, immutable, reviewed contract.
- **Boundary clarification.** The function does not block a prompt change on taste; the function blocks on contract violations (unsigned, unreviewed, missing eval verdict). A prompt change the function dislikes stylistically is still a valid candidate if the eval peer approves it.

### 3.2 The feature SLO contract (chapter 03)

- **Who owns.** Jointly authored by the function and the ML-platform peer; signed before a feature set enters the serving stack.
- **Function's role.** Specify the shape (availability, freshness, completeness, parity); hold the SLO values quarterly; operate the burn-rate alarms.
- **Boundary clarification.** The SLO values are negotiated — they reflect both the peer's pipeline capability and the function's reliability target. A unilateral tightening by either side is a change-control violation.

### 3.3 The eval-set release-gate contract (chapter 04)

- **Who owns.** The function authors the contract (chapter 04 §6); the eval peer signs the eval predicate and the thresholds; the mlops peer's pipeline enforces it.
- **Function's role.** Keep the contract current; coordinate the quarterly review; mediate between eval's and the function's thresholds.
- **Boundary clarification.** Demotion from `blocking` to `warning` is the function's unilateral action when the eval rots. Retirement of an eval set is the eval peer's. Change in threshold values is the eval peer's.

### 3.4 The retrieval probe set (chapter 02)

- **Who owns.** The rag-engineer peer authors the probes (content). The function operates the runner.
- **Function's role.** Schedule, time, alert on, and emit the `retrieval_recall_probe_hit_ratio` signal. Treat the probe set as a versioned artifact.
- **Boundary clarification.** The function does not add probes; the function adds signal emission. A probe-coverage gap identified on-call routes to the peer.

### 3.5 The single pane (chapter 05)

- **Who owns.** The function owns the pane; the peers subscribe to and read it.
- **Function's role.** Author the dashboard JSON, keep it in Git, run the weekly operational review, set the pane's own SLO.
- **Boundary clarification.** Peers may author deep-dive views (their own dashboards, their own vendor UIs); the on-call reads the shared pane first. Peer-authored views do not replace the shared pane.

## 4. Anti-patterns at the pipeline boundary

Each of these the function must recognise and prevent:

- **The function rewrites a feature pipeline's Flink job because "it keeps stalling."** The peer's domain expertise is replaced by the function's generalist fix; the pipeline becomes doubly-owned; the rollback path is unclear. Right action: fix the signal that shows the stall, route to the peer with evidence, measure the peer's response time against the SLO.
- **The function authors a prompt change under incident pressure.** An on-call page fires; the on-call "fixes" it by editing the prompt. There is no eval verdict; the audit trail shows an SRE as the author; the application team did not consent. Right action: the on-call rolls back the prompt (chapter 01 §6) — a reversion, not a new edit — and opens an incident with the application team.
- **The function chooses the next embedding model.** A retrieval performance regression is identified; the function picks a faster embedding model and ships it. The rag-engineer peer did not sign off; the eval against the new embedding model was never run. Right action: route to the peer with the evidence; let them propose; the function's release-gate signs off on their proposal.
- **The eval peer ships a new eval set without the function's awareness.** The release-gate pass rate shifts overnight; release velocity changes; the function investigates. Right action: eval-set changes flow through the release-gate contract revision process (chapter 04 §6.3).
- **Pipeline signals live only in a vendor UI.** Pinecone's dashboard, Tecton's UI, LangSmith's console each show a signal the function needs; none export it to Prometheus. Right action: require an exporter at onboarding; a signal that is not in the single pane is not a reliability signal.
- **Shared ServiceAccounts across the pipeline surfaces.** The feature-pipeline writer, the prompt-registry writer, and the release-gate updater all use one `pipeline-admin` identity. A misconfiguration on any one side affects all three; audit cannot distinguish sources. Right action: RBAC-split each peer's service into its own identity.
- **The single pane fills with noise nobody owns.** A peer adds a chart that fires false positives; the on-call starts ignoring the row. Right action: quarterly pane-audit removes unowned signals and the function names a specific peer owner per row.
- **Vocabulary drift across peers.** The rag-engineer peer says "freshness"; the ML-platform peer says "latency-of-materialisation"; the function's dashboards say "staleness." Three names for overlapping concepts. Right action: the quarterly cross-peer review reconciles.
- **Release-gate verdicts without eval-hash provenance.** A gate signs off against "the eval" without recording which eval-set hash. Later contamination is found in an older hash; the function cannot tell which verdicts need re-review. Right action: enforce the chapter 04 §6.2 verdict shape.

## 5. When the peer lines must still be read backwards

Boundary chapters sometimes over-fence. Four cases where the boundary bends:

### 5.1 The feature pipeline is stalled and the peer is unreachable (holiday, offboard, outage in the peer's tooling)

- **Pattern.** The pipeline has been stalled for 90 minutes; the feature-freshness SLO is about to burn fast; the ML-platform peer's on-call is not answering.
- **Function's role.** Execute the safest short-term action from `rb-feature-pipeline-stalled` — typically, degrade the feature-set to the last-known-good materialisation and route the serving tier to a fallback that does not depend on real-time features. Open an incident; keep the application team informed; do **not** modify the pipeline's code. When the peer is reachable, hand off with a full incident timeline.

### 5.2 The eval peer cannot produce a verdict fast enough for the release cadence

- **Pattern.** The eval pipeline runs too slowly; releases are held constantly; the function is pressured to loosen the gate.
- **Function's role.** Document the mismatch; propose either a faster eval subset for per-release gating (the eval peer owns the design of the subset) or a reduced release cadence (the application team owns that negotiation). The function does not loosen the gate unilaterally.

### 5.3 A retrieval incident requires a knob the rag-engineer peer has not exposed

- **Pattern.** The function needs to change the re-rank skip threshold to mitigate a latency burn; the current config does not expose it.
- **Function's role.** Work with the peer to expose the knob as config (chapter 01 §4); in the interim, degrade the fleet to a non-reranked path if `rb-rerank-latency` supports it.

### 5.4 A contamination signal fires with low confidence

- **Pattern.** Chapter 04's contamination score crosses a threshold but the eval peer is uncertain whether the signal is real.
- **Function's role.** Default to quarantine (chapter 04 §4.4). The function's bias is "cheap false positive, expensive false negative." The peer's investigation proceeds with the gate held.

In each case, the pattern is the same: the function identifies the cross-boundary signal, names the owning peer, holds the line on what the function does and does not own, and documents the workaround explicitly so it does not become permanent.

## 6. The function's irreducible pipeline-reliability surface

To restate what this module owns, now that the exclusions are explicit:

- **The prompt and config registry contract** (chapter 01 §3–§4), the progressive-delivery hook for prompt / config changes (chapter 01 §5), and the prompt-rollback runbook (chapter 01 §6).
- **The retrieval-freshness, retrieval-recall, embedding-version-mismatch, and re-rank-latency SLOs and signals** (chapter 02), and the four retrieval runbooks.
- **The feature-serving SLO set, the drift and pipeline-stall detectors, the parity audit, and the metadata-store SLOs** (chapter 03), plus the three feature runbooks.
- **The eval-set reliability signals (representativeness, discriminator, contamination, ceiling), the quarantine / demotion actions, and the eval-pipeline SLOs** (chapter 04), plus the three eval-related runbooks.
- **The single-pane observability dashboard, the Alertmanager wiring for pipeline SLOs, and the weekly operational review** (chapter 05).
- **The provenance trail** — every served response carries (`model_version`, `prompt_hash`, `config_hash`, `eval_hash`, `corpus`, `feature_set_version`, `channel`) — and the audit emission against `mod-110` retention.
- **The release-gate contract integration** for prompt / config / feature-set / eval-set candidates, composing with `mod-111` chapter 05.

Nothing above is co-owned without a named peer; everything above is accountable to the function.

## 7. The peer-boundary worksheet

Use this before new pipeline work is added to the module's scope and re-read it when a peer disagreement reaches the function's lead:

1. **Does this work implement a feature transformation, pipeline code, or feature-store operation?** If yes, it belongs to `ai-infra-ml-platform`.
2. **Does this work implement retrieval internals — chunker, embedding selection, index type, re-rank model?** If yes, it belongs to `rag-engineer`.
3. **Does this work author eval samples, judge prompts, calibration methodology, or benchmark selection?** If yes, it belongs to `ai-eval-engineer` or `model-evaluation-engineer`.
4. **Does this work write the CI/CD pipeline?** If yes, it belongs to `ai-infra-mlops`.
5. **Does this work implement the engine's prompt / config consumption semantics (how the engine reads them, how it applies decoding parameters)?** If yes, it belongs to `ai-infra-inference-engineer`.
6. **Does this work set retention or compliance policy on pipeline records?** If yes, it belongs to `mod-110`'s compliance discipline.
7. **Does this work author the prompt content itself?** If yes, it belongs to the application team (with eval-peer review).
8. **If none of 1–7 apply, is this work pipeline reliability — signal emission, SLO enforcement, release-gate integration, or on-call response?** If yes, it is this module's.

A pipeline-reliability program that can execute this worksheet every time its surface grows is a program whose boundary is honest. A program that cannot is a program that will drift into the peers' work and leave its own uncovered.

## Summary

- Four peers compose with this module: `rag-engineer` (retrieval authoring, level 25), `ai-infra-ml-platform` (feature authoring, level 30), `ai-eval-engineer` (eval authoring, level 30), and `model-evaluation-engineer` (eval methodology, level 30). Three more peers touch the module lightly: `ai-infra-mlops`, `ai-infra-inference-engineer`, and the application teams.
- The function explicitly does **not** author prompts, features, retrieval internals, eval content, or CI/CD pipelines. The function owns the reliability contract on what each surface exposes to serving and the on-call response when the surface's signals burn.
- Five surfaces have shared ownership — the prompt registry, the feature SLO contract, the eval-set release-gate contract, the retrieval probe set, and the single pane — and each has a clearly named producer / consumer split.
- Nine anti-patterns are named, each eroding the pane or the boundary: function-rewrites-pipeline, function-edits-prompt-in-incident, function-chooses-embedding-model, eval-ships-without-notice, vendor-UI-only signals, shared ServiceAccounts, unowned signals in the pane, vocabulary drift, verdicts without eval-hash provenance.
- When the boundary bends (peer unreachable, cadence mismatch, missing knob, low-confidence contamination), the function documents the workaround, routes to the owning peer, and does not fork the peer's work.
- The function's irreducible surface is the registry contract, the four surfaces' SLOs and signals, the quarantine / demotion authorities, the single pane, the provenance trail, and the release-gate integration.
- An eight-point worksheet decides, for any new pipeline work, whose surface it belongs to.

This closes module 112. The pipeline-reliability program it specifies is the practice that turns the module's chapters into a continuous operational discipline for the data and config surfaces that feed every AI serving path — and the peer boundaries it draws are what let the function compound its work rather than compete with the peers whose expertise the pipelines depend on.
