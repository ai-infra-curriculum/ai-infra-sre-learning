# Blameless Post-Mortems for AI Systems

## Why this chapter exists

The blameless post-mortem is the single most load-bearing cultural artifact of SRE. Google's SRE book defines it; John Allspaw's Etsy "Blameless PostMortems and a Just Culture" essay (2012) names it; the industry has spent over a decade refining the template. The template works: it names what happened, what contributed, what we learned, what we will change, and it does so without punishing the humans whose decisions were part of the chain.

The AI SRE inherits the template unchanged in spirit and adds three things to it:

- **AI-specific root-cause categories.** Classical SRE's root-cause taxonomy (software bug, config change, capacity, dependency failure) is incomplete for AI systems. Model-weight change, kernel regression, scheduler eviction, prompt / config drift, retrieval regression, provider degradation, agent cascade, eval drift — each is a distinct root-cause class that the template must name so that future post-mortems can be grepped by class.
- **Mandatory regression-fixture back-feed.** Every post-mortem produces at least one durable fixture — an alert, a canary case, an eval-set entry, a load-test shape, a chaos experiment, a runbook amendment — that would have caught the incident. Fixture back-feed is a required output of the retrospective, not a nice-to-have. Chapter 05 is dedicated to the discipline.
- **Cross-peer co-authorship when the root cause spans ownership.** AI incidents often sprawl across the serving layer, the application layer, the eval layer, and the vendor. A post-mortem authored by one engineer in one function undercounts the surface. The chapter codifies co-authorship when the incident touches more than one owner.

This chapter adapts the canonical template, specifies the AI-specific additions, and defines the authoring and review cycle the AI SRE function runs.

## 1. The blameless premise, restated for AI

Blameless means: we assume the humans involved did the best they could with the information and tools they had, and the post-mortem's object is the **system** — the tools, the signals, the runbooks, the training, the review processes — not the human.

Three AI-specific traps re-confirm why this matters:

- **Model behaviour is probabilistic.** An AI engineer who picked a prompt that performs well on 99.5% of inputs is not negligent when the 0.5% emits a toxic output. The post-mortem does not blame the engineer; it asks why the eval set did not have representative 0.5%-cases, why the moderation path did not catch it, and why the detection took as long as it did.
- **Vendor opacity.** When a model provider silently updates a model and quality regresses, the on-call engineer who ran the deploy is not the author of the regression. The post-mortem does not blame "someone should have noticed earlier." It asks why the canary eval cadence was slower than the vendor's update cadence, and whether version-pinning should be revisited (`mod-107` chapter 06).
- **Alignment and safety failures look like human failures.** When a model emits an inappropriate output, the instinct is to blame the engineer who wrote the system prompt. The blameless premise forces the inquiry past the proximate human into the system shape that permitted it — the safety filter configuration, the eval set gap, the deployment gate that let the version through.

The template below enforces blamelessness by making the question "what should the system have done differently" the primary structural element. If the first draft of a post-mortem names a person as the root cause, the IC / retro owner sends it back.

## 2. The template

The AI SRE function's template extends the Google / Allspaw shape with AI-specific sections. The structure is a hard contract so that a reader of any post-mortem finds the same information in the same places.

### 2.1 Header

- **Incident ID.** The incident channel's ID (`inc-2026-11-04-ttft-summariser`).
- **Title.** A one-line human-readable description.
- **Severity (final).** The severity the incident resolved at (often higher than at declare).
- **Start / detect / declare / resolve timestamps.** UTC.
- **TTD / TTR / TTM.** Time to detect (first signal to declare), time to mitigate (declare to stable), time to resolve (declare to resolve).
- **Authors.** The named retro owner plus any co-authors. Single-ownership on a multi-peer incident is a tell that the retro is incomplete.
- **Reviewers.** The peers who must sign off before the retro is marked complete (see §5).

### 2.2 Summary

A one-paragraph executive summary. The executive summary is the only thing most readers read; the post-mortem fails if the summary is not accurate on its own. Includes:

- What happened (user-visible symptom).
- Scope (how many users, which product surfaces, which routes).
- Primary cause class (one of the AI root-cause categories in §3).
- Mitigation that resolved it.
- One sentence of "here is the one fixture we are back-feeding to make sure this never happens the same way again."

### 2.3 Impact

- **User-visible impact.** What users experienced, with numbers.
- **SLO impact.** Error-budget burn by SLO, named. ("The chat-availability SLO burned X% of its 28-day budget.")
- **Revenue / contractual impact.** If measurable; otherwise the Comms Lead and finance quantify after.
- **Safety / policy impact.** If applicable — this is the category legal and policy will read.
- **Data / privacy impact.** If applicable.

### 2.4 Timeline

The timeline is a flat chronological list of events. Each event has UTC timestamp, actor (role, not name), event type (signal, decision, mitigation, communication), and a one-line description. The incident channel's scribed timeline is the source; the retro owner cleans and anonymises.

The timeline is not the place for interpretation. Interpretation goes in the contributing-factors section below.

### 2.5 Root-cause analysis

This is the structural heart of the post-mortem. Structure:

- **Primary cause.** One of the AI root-cause categories in §3. A sentence identifying the specific instance ("provider-side silent model-version update to gpt-4o-2024-11-20 degraded structured-output conformance on the `summariser` route").
- **Contributing factors.** A list of two to five. Each contributing factor is written in two parts — the fact, and the system-shape question it raises. "Canary eval cadence is daily; the provider update propagated in hours. This is a cadence mismatch that the SLO stack should close (§4 of `mod-105` chapter 03)."
- **Five-whys or equivalent.** The retro owner is encouraged to walk the five-whys or Ishikawa / fault-tree if the root cause is unclear, and to document the result. The method is less important than the discipline of not stopping at the first apparent cause.

### 2.6 What went well

The classical template section. For AI incidents, two typical entries:

- **Observability caught the signal.** Which specific signal (which metric, which runbook, which dashboard) and how early.
- **The fallback contract held.** Which route failed and which fallback absorbed the traffic (`mod-107` chapter 02).

Honest "what went well" matters because it names the controls that worked and must not be disassembled in the fix. A fix that improves one signal while accidentally disabling the signal that caught the incident is a regression.

### 2.7 What went poorly

- **Detection gaps.** What should have detected earlier.
- **Response gaps.** What took longer than it should have, with the system-shape question (runbook missing, SME unknown, scope-collection manual).
- **Communication gaps.** Named stakeholder who was not briefed on time.

### 2.8 Action items

The output of the retro. Each action item has:

- **Owner.** A named person on a named rotation. Not "the serving team."
- **Due date.** Explicit.
- **Severity.** P0 (production-blocking), P1 (required this quarter), P2 (nice-to-have).
- **Fixture type.** Alert / canary / eval / load-shape / chaos / runbook / documentation. Action items that are not fixtures ("talk to the vendor") are tracked, but they do not count toward the fixture back-feed quota.
- **Verification.** How we will know the action item has landed. "The alert is deployed and has fired at least once in a staging drill."

### 2.9 Fixture back-feed (mandatory)

A distinct section that lists each new or updated fixture produced by this incident, with the system it lives in. The schema is covered in chapter 05 §3; the shape is:

```yaml
fixtures_added:
  - kind: canary_eval_case
    system: evals/summariser/structured_output.jsonl
    description: adds the exact failing shape from this incident
    owner: ai-eval-engineer
  - kind: alert
    system: prometheus
    name: structured_output_validator_error_rate_high
    threshold: 3× 7-day baseline, 10m window
    runbook: rb-structured-output-break
    owner: ai-infra-inference-engineer
fixtures_updated:
  - kind: runbook
    name: rb-structured-output-break
    change: added vendor-side-mode-regression confirm step
    owner: ai-sre
```

If this section has no entries, the retrospective is not complete. The retro owner either lists the fixtures or documents why no fixture is possible (which is a rare and reviewable claim).

### 2.10 Related incidents and prior art

- **Repeat events.** Links to past incidents whose root-cause class matches.
- **Related runbooks and chapters.** `rb-X`, `mod-105 chapter N`.
- **External references.** Public post-mortems that match the shape (chapter 04 of this module).

### 2.11 Appendix

Supporting artifacts — screenshots, trace IDs, dashboard panels, SME write-ups, raw timeline, bridge-call notes. The appendix is referenced by the body; it is not expected to be read linearly.

## 3. The AI-specific root-cause categories

The post-mortem's primary-cause field is drawn from a controlled vocabulary. The eight categories below cover the surface this module teaches; each is tied to the `mod-105` taxonomy so that the two modules share vocabulary.

### 3.1 Model-weight

A change in model weights — a provider-side silent model-version update, an in-house fine-tune promotion, a weight-corruption event, a progressive-rollout that landed a regression. Includes the "silent output drift" sub-class of `mod-105` chapter 03.

Typical contributing factors: version not pinned; canary eval cadence longer than the drift horizon; checksum / signature verification disabled.

### 3.2 Kernel

A GPU kernel, driver, CUDA, cuDNN, FlashAttention, NCCL, or numeric-format change. The surface Pytorch / Triton / XLA / TensorRT compilation sits in. Includes the "kernel regression" sub-class of `mod-105` chapter 03.

Typical contributing factors: driver upgrade without canary; numeric-format silently changed on a model-swap; XID error class interpreted as benign when it was not.

### 3.3 Scheduler / infrastructure

Kubernetes scheduler, autoscaler, preemption, PDB, priority-class, node pool, cluster-autoscaler. GPU preemption-adjacent (`mod-105` ranges over this in chapter 03's "scheduler eviction" sub-class).

Typical contributing factors: PDB too low; priority class misconfigured; spot volatility exceeded the model's capacity recovery; batch job scheduled against serving pool.

### 3.4 Prompt / config

A prompt template, system prompt, agent scaffolding, feature flag, model-id config, or application code change. The application-layer surface the AI engineer owns.

Typical contributing factors: prompt change invalidated cache; schema change did not propagate to validators; feature flag defaulted on in prod.

### 3.5 Retrieval

Index staleness, embedding drift, embedding-model change, chunking change, tokenizer change, ingest failure, index-replica desynchronisation. The retrieval surface (`mod-105` chapter 05).

Typical contributing factors: embedding-model version changed silently; ingest lag alert threshold not tight enough; chunk-size change invalidated the vector space.

### 3.6 Provider

A hosted model / embedding / moderation provider outage, degradation, model deprecation, breaking API change, or silent behavioural change. The vendor surface (`mod-107` chapter 03).

Typical contributing factors: fallback contract did not cover the route; status-page ingestor lagged; API-version not pinned.

### 3.7 Agent cascade

A multi-step agent, orchestrator, tool-call pipeline, or planner feedback loop. The agent surface (`mod-105` chapter 06).

Typical contributing factors: max-depth / max-cost cap not enforced; tool returned a shape the agent retried into; planner version changed and loop shape changed.

### 3.8 Eval / judge drift

A judge-model, canary-eval-set, or eval-pipeline change — the measurement itself regressed. The eval surface (`mod-105` chapter 07).

Typical contributing factors: judge model-version drift invalidated thresholds; eval-set contamination from training data refresh; calibration set stale.

A ninth category exists but is reserved for the "we could not classify this" case:

### 3.9 Unknown / multi-class

Used sparingly and reviewed closely. The retro owner must document both what they tried and why the classification resists a single category. Unknowns are a useful signal when they accumulate — the AI SRE function tracks the ratio and invests in better instrumentation when it grows.

## 4. Writing the post-mortem

### 4.1 Who authors

The retro owner is assigned at resolve by the IC. The default heuristic:

- **Primary cause = model-weight / kernel / scheduler / provider:** the AI SRE on-call or a named `ai-infra-performance` / `ai-infra-platform` peer.
- **Primary cause = prompt / config / agent cascade:** `AI Engineering` with AI SRE co-author.
- **Primary cause = retrieval / embedding:** `ai-infra-pipeline` with AI SRE co-author.
- **Primary cause = eval / judge drift:** `ai-eval-engineer` with AI SRE co-author.

The AI SRE's irreducible role is the **composition oversight** — the post-mortem of an incident that spans multiple surfaces is AI SRE co-authored whether the primary cause lives in a peer's domain or not.

### 4.2 Co-authorship when the root cause spans ownership

Many AI incidents have a chain: the provider deployed a new model, the embedding-model cadence changed with it, the retrieval regressed, the agent planner behaviour changed, the cost spiked. A single-owner retro will pick one link and call it the cause. The co-authorship rule:

- **If the timeline shows the incident crossed two or more root-cause categories**, two co-authors are named, one from each primary peer.
- **The AI SRE is a co-author by default** for any incident that spans more than one category, because the composition is this role's domain.
- **Co-authors draft in parallel** on the same document; the retro owner gates merge and submission.

### 4.3 Drafting discipline

- **Write in past tense, third-person, factual voice.** "The deploy at 13:05Z increased `inference_ttft_seconds` p95 to 4.8s." Avoid "we accidentally broke" and similar constructions.
- **Use actor roles, not names.** "The on-call engineer" not "Sam." Names appear in the author list only.
- **Attach artifacts to the appendix as references**, not inline. The body stays short enough to read in ten minutes.
- **Separate observation from inference.** Observation: "The structured-output validator error rate rose to 11%." Inference: "This is consistent with a vendor-side structured-output mode regression." The template keeps these in different sections so that the inference can be contested without rewriting the observation.
- **Say "we do not know" when you do not know.** The retro is not a demonstration of competence; it is a record of what was learned.

### 4.4 Deadlines

The retro is drafted within **two business days** of resolve and submitted within **five business days**. The two-day draft is non-negotiable because memory decays; the five-day submission allows co-authorship and review.

The AI SRE function's leadership sees the submission dashboard. An incident without a filed post-mortem at the five-day mark surfaces on a leadership weekly report. Over time the organisation's retro culture is visible in that dashboard.

## 5. The review cycle

A completed draft goes through a formal review. The AI SRE function's review cycle:

### 5.1 Peer-SME review

The named peer whose surface was the root cause reads the draft and signs off. Sign-off is a channel comment or a GitHub review. Disagreement is in-document; the retro owner resolves or the IC arbitrates.

### 5.2 AI SRE function review

An AI SRE other than the retro owner reviews for:

- Template compliance (all sections present, header complete).
- Primary-cause category assignment reasonable.
- Fixture back-feed present and non-empty.
- Co-authorship present for multi-category incidents.
- Blameless tone — no proximate-human blame language.
- Action items have owners and due dates.

### 5.3 Legal / policy review (triggered)

For post-mortems touching `rb-toxic-output-blast`, `rb-weight-corruption` serving at scale, safety-filter regressions, eval-set poisoning, or regulated-domain impact, legal and the `ai-risk-engineer` function review before publication. They may request redactions, non-publication, or a separate formal-notification narrative (chapter 06). The retro owner respects their call.

### 5.4 Publication

Internal publication is the default. Post-mortems are published to a searchable internal repository, indexed by incident ID, primary-cause category, and date. An "external" track exists for incidents the organisation chooses to publish publicly (the OpenAI / Anthropic / Google / Microsoft engineering-blog post-mortems analysed in chapter 04 are examples); that track adds PR and legal gates.

## 6. Follow-through

The post-mortem is useless if its action items rot. The AI SRE function tracks:

- **Action-item completion rate** by severity, by category, by owner.
- **Fixture activation.** Did the alert fire? Did the canary eval detect its first synthetic injection? Did the chaos experiment run? A fixture that was never exercised is worth less than it claims.
- **Repeat-incident rate.** How often the same root-cause category recurs in the following quarter. A category with a rising repeat rate is a signal that the fixtures being produced are not the right fixtures.

The metrics live on the AI SRE function's weekly dashboard alongside the SLO burn-rate panels.

## 7. Trap patterns to avoid

Six failure modes recur. The retro owner (and reviewer) is explicit about catching them:

- **The convenient root cause.** Picking the proximate deploy-shaped change as the cause when the real cause is a longer-horizon drift. Common in provider / embedding / judge-drift incidents.
- **The "process" action item.** "We will be more careful" is not a fixture. If the action item does not change the code / config / tooling / runbook / eval / alert in a way an auditor can observe, it is cosmetic.
- **The "educate the team" action item.** Education is not a fixture. If the system requires humans to remember a thing to avoid a recurrence, the system is not fixed.
- **The private channel.** Drafting in a DM instead of in the review repository, because the author is self-conscious about the draft. The function's culture is explicit: drafts are visible; reviewers help.
- **The repeat-victim.** When the same engineer is "the one who did the thing" twice, the retro reviewer looks at the system shape that keeps assigning them to that surface, not at the engineer.
- **The over-redacted retro.** Legal requests redaction; the retro owner redacts the entire technical narrative "to be safe." The resulting document teaches nothing. The retro owner negotiates specific redactions, not blanket ones.

## 8. The AI SRE's post-mortem worksheet

1. **Open the template** at the resolve-channel post; assign the retro owner and co-authors.
2. **Pin the timeline source** (the incident channel's pinned message / document).
3. **Classify the primary cause** from the controlled vocabulary in §3.
4. **Walk the five-whys** or equivalent for contributing factors.
5. **Pick at least one fixture back-feed.** Name the system, the owner, and the verification.
6. **Write in past tense, actor-role voice, with observation and inference separated.**
7. **Submit the draft within two business days.**
8. **Route through peer-SME, AI SRE, and (if triggered) legal review.**
9. **Publish; file the action items with owners and due dates.**
10. **Track completion and repeat-rate on the dashboard.**

## Summary

- **The template is Google / Allspaw plus AI-specific additions.** Root-cause vocabulary, fixture back-feed, co-authorship on multi-surface incidents.
- **Blameless means system-shape questions, not person-shape blame.** The retro owner and reviewer are explicit about catching proximate-human language.
- **The primary cause is drawn from eight categories** — model-weight, kernel, scheduler, prompt / config, retrieval, provider, agent cascade, eval / judge drift — that align with `mod-105` and with the runbook library.
- **The fixture back-feed is mandatory.** No post-mortem is complete without at least one named, owned fixture entering the SLO stack.
- **Co-authorship on multi-peer incidents is non-negotiable.** The AI SRE is a default co-author because composition oversight is this role's domain.
- **Timing matters.** Two-day draft, five-day submission, tracked at the function-leadership weekly cadence.

Chapter 04 shifts from the function's own retros to the discipline of reading the industry's: how to dissect public post-mortems from the frontier labs and extract the shape that this role is hired to prevent.
