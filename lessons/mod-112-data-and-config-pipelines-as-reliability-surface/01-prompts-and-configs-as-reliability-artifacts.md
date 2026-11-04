# Prompts and Configs as Reliability-Critical Artifacts

## Why this chapter exists

On a classical web service, the shipping artifact is a container image. Config is tracked but secondary, and prompts do not exist. On an AI service, three distinct artifacts flow into every response — the **model weights**, the **prompt template**, and the **runtime config** (decoding parameters, routing rules, retrieval settings, safety-guardrail thresholds, feature flags). The function in `mod-111` wrote the release-engineering discipline for the first of these. This chapter writes the equivalent discipline for the other two.

The temptation on most teams is to treat prompts and configs as "just strings" — edited in a Google Doc, pasted into a dashboard, hot-updated from a UI. A prompt change ships faster than a code change, with less review, with no release gate, and often with no audit trail. The payoff is velocity. The cost is that the two most common causes of a user-visible AI regression — a quiet prompt tweak and a decoding-parameter drift — are the two changes the function has the least ability to see, let alone roll back.

The chapter's claim is strict: **prompts and runtime configs are first-class deploy artifacts and must flow through the same progressive-delivery stack as model weights**. Not the same tooling pretending to apply to a different artifact; the literal canary, shadow, blue-green, auto-rollback machinery from `mod-111`, with a few shape adjustments for the fact that prompt rollouts have different cadence and blast radius characteristics. The boundary of this chapter is the shape of the artifact and the operational contract. What goes *inside* the prompt — the craft of prompt engineering — is at the `ai-eval-engineer` peer and the application team; the function does not edit the rubric.

Primary references: [Microsoft Prompt flow](https://microsoft.github.io/promptflow/), [LangSmith prompt management](https://docs.langchain.com/langsmith/create-a-prompt), [PromptLayer](https://docs.promptlayer.com/), [Agenta](https://docs.agenta.ai/), [Helicone](https://docs.helicone.ai/), the [OpenFeature specification](https://openfeature.dev/specification/), and [Flagger](https://docs.flagger.app/) / [Argo Rollouts](https://argo-rollouts.readthedocs.io/) from `mod-111`.

## 1. The artifact taxonomy

An AI SRE function with a coherent prompt / config discipline begins by naming what, precisely, ships. The term "prompt" in practice covers several distinct objects; the function needs separate lifecycles for each.

### 1.1 The prompt template

A **prompt template** is a structured artifact that produces a request body when parameters are filled in. It typically includes:

- A **system prompt** — the role, tone, and behavioural priors the model is instructed with.
- **Few-shot exemplars** — zero or more input / output pairs demonstrating the expected shape.
- **Variable placeholders** — slots the application fills (user message, retrieved context, tool-call history).
- **Output-format instructions** — JSON schema, tool-choice constraints, stop sequences.

A template is identified by a content hash. The content hash is the deploy key. Two templates that differ by a single whitespace are two distinct artifacts; treating them as "the same prompt, minor edit" is where most audit-trail corruption begins.

### 1.2 The runtime config

A **runtime config** is the per-request or per-route settings that select *how* the model is invoked:

- **Decoding parameters** — `temperature`, `top_p`, `top_k`, `max_tokens`, `frequency_penalty`, `presence_penalty`, stop sequences.
- **Routing rules** — which model is selected for a given route, tenant, cohort, or prompt-length band; which provider is primary vs. fallback.
- **Retrieval settings** — `top_k`, re-rank threshold, context-window-fill ceiling, retrieval-freshness bound (chapter 02).
- **Safety and guardrail thresholds** — classifier confidence cutoffs, refusal policies.
- **Feature flags** — the on/off switches that gate experimental behaviour.

Runtime config is a deploy artifact in the same sense that a serving-tier NGINX config is. A change to `temperature` from `0.1` to `0.7` is as consequential as a code change to the sampling loop. The function treats it that way.

### 1.3 The combined deploy unit

The release-gate contract (`mod-111` chapter 05) evaluates a candidate that is the *product* of a model-weight pin, a prompt-template hash, and a runtime-config hash. A promotion decision must reference all three. Shipping the model and leaving the prompt and config free-floating is the anti-pattern; the audit log must link every served response to the specific (model, prompt, config) triple that produced it.

### 1.4 What this chapter is not

- **Prompt-engineering craft.** Chain-of-thought vs. direct answer, structured output, prompt compression, retrieval-prompt stitching — all at the `ai-eval-engineer` peer and the application team. The function wires the pipeline that ships prompts; the function does not author the prompt.
- **Config schema design per provider.** Each inference engine and provider has its own parameter space. The function enforces that the chosen parameters are versioned; the engineer peer owns which parameters to expose.

## 2. Why "just strings" is a reliability problem

Before writing the discipline, the function must name the failure modes it is solving. Each is drawn from operational experience and each explains why the surrounding progressive-delivery stack is needed.

### 2.1 The silent prompt edit

A product manager edits the system prompt in the LLM-trace vendor's UI to "soften the refusal tone." No commit lands. No review happens. No rollout controller gates it. The change takes effect at the next request. Two hours later the on-call pages on a judge-score regression and a refusal-rate spike. The root cause is a prompt edit that the on-call has no way to see, because the prompt is not under version control at all.

The mitigation is §3's prompt registry: a Git-backed artifact store from which the serving tier reads prompts by content hash. The vendor UI is read-only; the write path is a PR.

### 2.2 The decoding-parameter drift

A team adopts a new routing library that changes the default `temperature` from `0` to `1` when the parameter is not explicitly set. A service that previously ran at `temperature=0` now runs at `temperature=1` on half its routes because of a library upgrade. Judge scores shift, output lengths grow, costs spike. No deploy was tagged as changing the model; the library upgrade is in a different repo.

The mitigation is §4's config-as-manifest discipline: every runtime parameter has an explicit default in the service's own manifest, and no library default is relied on silently. The service refuses to start if a required config key is missing rather than falling back to a vendor default.

### 2.3 The prompt and model co-change

The eval peer signs off on a new model only *with* a new prompt. The mlops peer ships only the model; the prompt that was validated against it never ships. The gate's eval verdict was against the (model-v2, prompt-v2) pair; production is now running (model-v2, prompt-v1). The gate does not know; the audit trail does not know.

The mitigation is §1.3's combined deploy unit: the release-gate verdict is keyed on the (model, prompt, config) triple, and the serving tier rejects any triple whose components are not jointly signed.

### 2.4 The rollback that cannot roll back

A quality regression fires. The on-call rolls back the model. The regression persists. The actual bad change was a prompt edit that landed in parallel, through a different pipeline; the model rollback does not touch it. Minutes pass. The on-call does not have a "roll back the prompt" button, because prompt changes were never a release artifact to begin with.

The mitigation is §5's prompt rollback as a first-class reliability operation: the function has a runbook, a controller action, and a time-to-rollback target for the prompt artifact just as it does for the model.

## 3. The prompt registry

The foundational piece is a **prompt registry** — a content-addressed store of prompt templates with a versioned audit log. There are two acceptable implementations: a dedicated prompt-management platform (LangSmith, PromptLayer, Agenta, Promptfoo, Helicone, Microsoft Prompt flow) and a Git-native implementation (prompts as files in a repo, released through the same CI/CD plumbing as code). The function picks one; the discipline is identical.

### 3.1 The registry contract

Regardless of implementation, the registry must satisfy:

- **Content-addressed.** A prompt is identified by a hash of its canonical serialisation (whitespace-normalised, variable-name-normalised, exemplar-order-stable). The hash is the only key the serving tier uses.
- **Immutable.** A prompt at a given hash is never mutated. An "edit" produces a new hash; the old hash stays resolvable forever.
- **Audit-logged.** Every write records the author, the time, the preceding hash, the PR or ticket, and the review state. The eval verdict against this prompt is linked.
- **Reviewed.** No write to the registry lands without a review from the application owner and, when the change affects a quality-SLO route, the eval peer.
- **Served by API.** The serving tier resolves `prompt_hash -> template body` through a cacheable API call, not by embedding the body in the deploy image. This is deliberate — it allows a prompt rollback without a code redeploy.

### 3.2 Platform vs. Git-native

Both approaches implement the contract; the tradeoff is on the audit and review surface.

| Axis | Prompt-management platform | Git-native prompts |
|------|---------------------------|--------------------|
| Review surface | Platform UI; depends on platform's PR model | Standard code review |
| Audit log | Platform-owned; retention per platform policy | Git history + retention policy at `mod-110` ch. 05 |
| Eval integration | Often native (platform runs evals against prompts) | Requires a separate eval harness wired to Git-tracked prompts |
| Rollback granularity | Prompt-only rollback via platform API | Prompt-only rollback via config-only deploy |
| Multi-tenancy of edits | Platform ACLs | Codeowners |
| Risk | Vendor lock-in on the audit trail | More plumbing to build the CI/CD into |

The function's default on a well-governed team is **Git-native prompts** — prompts as files under `prompts/<route>/<name>.yaml`, each with frontmatter capturing the eval-set reference and the owner, rendered to the registry API as part of CI. The Git-native shape makes the prompt indistinguishable from any other deploy artifact, which is the point. A team with a mature prompt-management platform already adopted can also satisfy the contract through the platform, provided the platform exposes an export / audit path that satisfies `mod-110`'s retention discipline.

### 3.3 The serving tier's read path

A typical serving flow:

```
request
  -> route resolver (which template for this route?)
  -> registry client: GET /prompts/<hash> (cached locally, TTL = minutes)
  -> template renderer: substitute variables
  -> inference client: call model with rendered body
  -> emit response, tag with (model_version, prompt_hash, config_hash)
```

The `prompt_hash` is stamped on every response, log, trace, and metric. It joins `model_version` as a first-class dimension of the SLI stack (`mod-104` chapter 01). A promotion is identified by the triple; a rollback targets one component of the triple without disturbing the others.

### 3.4 The registry API

The registry exposes, at minimum:

- `GET /prompts/<hash>` — resolve a prompt by content hash.
- `GET /prompts/<route>/<channel>` — resolve the *currently-promoted* prompt for a route and channel (`stable`, `canary`, `shadow`). This is what the serving tier calls on a cold cache.
- `POST /prompts` — register a new prompt (invoked by CI on a merge).
- `POST /prompts/<route>/<channel>/promote` — rebind a channel to a hash (invoked by the rollout controller).
- `GET /audit/<hash>` — return the audit record for a prompt.

The registry service is a dependency the function operates. It gets its own SLO (`mod-103`), its own alert rules (`mod-104`), and its own runbook. A prompt registry that is down means the serving tier cannot resolve new prompts; the tier must cache the last-known-good resolution with a conservative TTL so a registry outage does not become a serving outage.

## 4. Config as manifest

Runtime config is handled identically in principle and with slightly different plumbing in practice. The Kubernetes-native shape is the shortest path.

### 4.1 Config as ConfigMap + CRD

Runtime config lives in a `ConfigMap` (per environment, per fleet) with a schema validated by a custom resource definition. The serving tier mounts the ConfigMap read-only; the serving process re-reads on a SIGHUP or on a watcher. A sample config:

```yaml
apiVersion: config.inference.io/v1
kind: RouteConfig
metadata:
  name: chat-route
spec:
  model: pin://anthropic/claude-3-5-sonnet-20241022
  prompt: hash://sha256:a7f3...e21c
  decoding:
    temperature: 0.2
    top_p: 0.95
    max_tokens: 2048
    stop: ["</answer>"]
  retrieval:
    top_k: 8
    rerank_threshold: 0.35
    freshness_bound_hours: 24
  safety:
    classifier_min_confidence: 0.80
    refusal_policy: "soft"
```

Each field is validated by the CRD's OpenAPI schema. Missing required fields are a hard failure at admission; silent defaulting is forbidden.

### 4.2 Why a CRD and not a plain file

The CRD gives:

- **Schema enforcement** at `kubectl apply` / admission time.
- **RBAC** per field class (only the SRE function can edit `retrieval.freshness_bound_hours`; only the application owner can edit `decoding.temperature`).
- **Audit** via Kubernetes' standard audit log, routed per `mod-110` chapter 05.
- **GitOps compatibility** out of the box (Argo CD, Flux).

A plain YAML file in a Git repo achieves most of this, but the CRD is the version that composes with the admission controllers at `mod-110` and the audit retention at `mod-110`. The function's default is the CRD shape.

### 4.3 Config-as-code boundaries

Feature-flag-shaped config (short-lived, high-churn, per-tenant) is different from route-shaped config (long-lived, global, per-route). The function uses:

- **OpenFeature-shaped flags** for experimental, per-tenant, high-churn switches. Flag changes go through the flag platform's audit log (LaunchDarkly, Unleash, Flagd); the flag platform's reliability becomes a dependency (`mod-107` chapter 04).
- **CRD-shaped config** for long-lived, promotion-gated settings. Changes go through the release pipeline.

Mixing the two shapes is where drift begins — a feature flag that was meant to be temporary becomes a permanent part of the routing config, but it lives in the flag platform and not the CRD. The function reviews flag-vs-config placement quarterly.

## 5. Prompt and config changes through progressive delivery

The contract: **a prompt or config change is a candidate just as a model-weight change is a candidate**, and it flows through the same Flagger / Argo Rollouts progressive-delivery machinery from `mod-111`.

### 5.1 The candidate shape

A candidate submitted to the release gate (`mod-111` chapter 05) is a triple:

```
candidate:
  model: pin://anthropic/claude-3-5-sonnet-20241022
  prompt: hash://sha256:a7f3...e21c
  config: hash://sha256:91b0...4d2f
```

Any one of the three may be the component that changed. The gate evaluates the whole triple against the eval predicate and the reliability predicate. A prompt-only change still runs the full eval; a config-only change still produces a candidate hash that the gate signs off against.

### 5.2 Canary for prompt and config

The same Flagger `Canary` resource from `mod-111` chapter 01 §7 applies — the serving tier reads the prompt and config from the channel the controller sets. The canary shifts traffic between a stable channel and a canary channel; each channel resolves to its own (prompt_hash, config_hash) at request time.

The AI-adapted analysis signal set (`mod-111` chapter 01 §3) applies unchanged: serving-health, quality, cost, business. The quality signal is often *more* sensitive to a prompt change than to a model change — prompt edits can alter refusal rate, response length, or format-conformance by a factor of two without any model change. The function adjusts the dwell windows upward on prompt-only canaries because the signal moves faster than the aggregate SLO alarm.

### 5.3 Shadow for prompt and config

Shadow traffic (`mod-111` chapter 03) is cheaper than a canary for prompt changes because the serving tier can render both the stable and candidate prompts from the same model backend — no second deploy is required if the model is the same. A prompt shadow is often the most cost-effective way to see whether a prompt edit shifts the response distribution before a canary begins.

The implementation: the serving tier resolves `prompt_hash` for both `stable` and `shadow` channels, invokes the model twice (once with each rendered body), returns the stable response to the client, and emits the shadow response to the trace store with a `shadow_of` correlation. The shadow's cost is the second model call; for cheap models this is tolerable, for expensive models the shadow sample rate is tuned.

### 5.4 Blue-green for config

Config changes that cannot be partially rolled out — a routing-table swap, a retrieval-provider swap — use blue-green (`mod-111` chapter 04). The green environment loads the new config; a routing cutover shifts all traffic at once; the blue environment is retained for the rollback window.

### 5.5 The rollout controller reads the channel

The Flagger / Argo Rollouts resource is extended to read from the prompt registry / config CRD rather than from a container image tag. For Flagger:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: chat-prompt-rollout
spec:
  provider: istio
  targetRef:
    apiVersion: config.inference.io/v1
    kind: RouteConfig
    name: chat-route
  analysis:
    interval: 2m
    threshold: 5
    stepWeight: 5
    maxWeight: 50
    metrics:
      - name: ttft-p95
        templateRef: { name: ttft-p95-vs-baseline }
        thresholdRange: { max: 1.15 }
      - name: judge-score-delta
        templateRef: { name: judge-score-delta }
        thresholdRange: { min: -0.05, max: 0.1 }
      - name: refusal-rate-delta
        templateRef: { name: refusal-rate-delta }
        thresholdRange: { min: -0.1, max: 0.1 }
      - name: response-length-delta
        templateRef: { name: response-length-delta }
        thresholdRange: { min: -0.2, max: 0.2 }
```

Flagger does not ship a `RouteConfig`-aware primary / canary primitive out of the box; the pattern above requires either a custom target-ref integration (an operator the SRE function writes that watches a `RouteConfig` with `stable` / `canary` sub-sections and manipulates the service mesh accordingly) or an Argo Rollouts `Rollout` whose `template` references the channel at the `env` layer. <!-- needs-research: verify the exact extensibility shape for Flagger custom target references against the current Flagger CRD documentation and cite the specific API field --> The chapter asserts the pattern; the implementation detail resolves at the SRE / mlops handoff.

### 5.6 Response-length as a first-class canary signal

On prompt and config canaries specifically, `response_length_distribution{route, channel}` deserves promotion from "monitor" to "gate." A prompt that silently inflates response length by 50% changes cost, latency, and user experience together; the three signals catch the regression individually, but the response-length signal catches it earliest. The canary analysis metric list includes `response-length-delta` as a required predicate.

## 6. Prompt rollback as a first-class reliability operation

The function treats prompt rollback with the same discipline as model rollback.

### 6.1 The rollback contract

A prompt rollback is:

- **One action** — a single CLI invocation or controller call rebinds the `stable` channel to a prior `prompt_hash`.
- **Time-bounded** — the function commits to a time-to-rollback target. A reasonable target for a mature team is 2 minutes from decision to full-traffic rebind; chapter 05 writes the SLO on the rollback operation itself.
- **Atomic with respect to serving** — no partial state where some pods serve the new prompt and some the old one for longer than the ConfigMap-propagation window.
- **Audit-logged** — the rollback is recorded in the same audit stream as a forward promotion.
- **Runbook-backed** — `rb-prompt-rollback` is a named, tested runbook. The DR drill program (`mod-113`) exercises it on a cadence.

### 6.2 Rollback is not a code deploy

The failure mode the contract prevents is "to roll back the prompt we need to cut a release of the serving container." That coupling makes the rollback slow enough that it is not usable in incident time. The registry's content-addressed, channel-bound design makes rollback a config edit, not a code edit:

```
$ promptctl rollback --route chat --to sha256:a7f3...e21c
Current: sha256:1d49...b8c7 (promoted 2026-11-03T14:22Z)
Target:  sha256:a7f3...e21c (previously stable 2026-11-02T09:11Z)
Rebinding chat/stable -> a7f3...e21c ...
Rollout controller: shifting 100% to a7f3...e21c
Audit: logged to compliance-store (retention: 2y per mod-110 ch. 05)
Done in 47s.
```

The time dominates the ConfigMap propagation + the service mesh traffic shift. Both are tunable; the function sets the target by measurement and holds the rollback drill accountable.

### 6.3 Who may invoke it

The rollback is a release-engineering operation, not a product operation. The on-call engineer invokes it from the runbook on a page; the application owner may not invoke it unilaterally during an incident (the on-call owns the incident per `mod-108`). Outside incidents, the rollback is subject to normal change control.

### 6.4 The rollback drill

Once per quarter, the function runs a prompt-rollback drill as part of the `mod-109` chaos program:

- A synthetic regression is introduced by promoting a known-bad prompt in a canary stream.
- The auto-rollback triggers; the function measures time-to-rollback.
- If the trigger does not fire, the drill fails; the alerting path is fixed before the next drill.

The drill's signal feeds the DORA MTTR metric (`mod-111` chapter 06) for the prompt-change class.

## 7. Audit, retention, and provenance

Every served response is tagged with:

- `model_version` — the pinned model identifier.
- `prompt_hash` — the content hash resolved at request time.
- `config_hash` — the hash of the serialised (temperature, top_p, top_k, routing, retrieval, safety) tuple in effect.
- `channel` — `stable`, `canary`, or `shadow`.

The tuple is emitted on the OTel trace (`mod-104` chapter 01) and the Prometheus `version` label (`mod-104` chapter 02). The compliance team's retention policy from `mod-110` chapter 05 governs how long the audit log is retained; the SRE function ensures the emission path is reliable, not what the retention is.

A later incident investigation (`mod-108` chapter 03) that pulls a response from logs can therefore reconstruct exactly which (model, prompt, config) triple produced it. Without this provenance, root-cause analysis on prompt / config regressions collapses to "someone edited something somewhere."

## 8. The prompt / config release checklist

Before enabling the prompt / config progressive-delivery stack on a route for the first time, the function confirms:

1. **The prompt registry is in place** and the serving tier resolves by content hash.
2. **The runtime config is in a CRD-backed ConfigMap** with schema validation at admission.
3. **The serving tier tags every response** with (`model_version`, `prompt_hash`, `config_hash`, `channel`).
4. **The release-gate contract** (`mod-111` chapter 05) is extended to accept the triple as the candidate and to invoke the eval predicate against it.
5. **Flagger / Argo Rollouts** is wired to read channel bindings from the registry and CRD.
6. **The four canary analysis predicates** (TTFT delta, judge-score delta, refusal-rate delta, response-length delta) are in Git and resolve to Prometheus queries with the `version` label populated.
7. **The `rb-prompt-rollback` runbook** exists, is linked from the incident board, and has been drilled.
8. **The audit emission** is validated against the retention target from `mod-110` chapter 05.
9. **The peer boundary is signed** — the eval peer owns prompt craft, the application team owns the prompt itself, the SRE owns the release and rollback pipeline. Chapter 06 writes the full worksheet.

A prompt / config release that runs without this checklist is an undefined operation waiting to produce an incident the function cannot investigate.

## Summary

- Prompts and runtime configs are first-class deploy artifacts. The two most common causes of a user-visible AI regression are silent prompt edits and decoding-parameter drift, and both must flow through the same release discipline as model-weight changes.
- Prompts live in a content-addressed registry (Git-native by default, or a dedicated prompt-management platform that exposes an equivalent audit path). Configs live in a CRD-backed ConfigMap with schema validation at admission.
- Every served response is tagged with the (`model_version`, `prompt_hash`, `config_hash`, `channel`) triple. Promotions are keyed on the triple; the release gate evaluates the triple as a unit.
- Prompt and config changes flow through the canary / shadow / blue-green patterns from `mod-111` unchanged; the AI-adapted analysis signal set is extended with `response-length-delta` as a first-class predicate because prompts move that signal earliest.
- Prompt rollback is a first-class reliability operation with a named runbook (`rb-prompt-rollback`), a time-to-rollback SLO, a drill cadence, and an audit trail. It is executed without a container redeploy.
- A nine-point checklist governs enabling the stack on a new route; a release outside the checklist is not covered by the function's reliability contract.

Chapter 02 moves from the control-plane artifacts to the data plane, writing the retrieval-freshness SLOs that the RAG routes depend on.
