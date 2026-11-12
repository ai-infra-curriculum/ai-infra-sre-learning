# Pattern-Matching Against Public Post-Mortems

## Why this chapter exists

A reliability engineer's library is not only their own retrospectives. The industry publishes AI-system post-mortems — some as formal engineering-blog posts from the frontier labs, some as status-page history, some as incident disclosures from the cloud providers that host the models, some as independent reporting of safety events that the vendor has implicitly or explicitly acknowledged. Reading them well is a skill. Reading them well **as an AI SRE** — not as a reader of tech news, not as a vendor evaluator, but as someone whose next week's work is shaped by the shape of the failure — is the skill this chapter teaches.

The ROI is high and non-obvious. A single frontier-lab post-mortem names the exact failure the authors were surprised by, the signal that caught it (or did not), the time-to-mitigate they achieved, the runbook they wrote in response, and often a sentence of "we are rolling out additional canary evals / rollback drills / telemetry." That sentence is the shape of a durable change in how they operate. Reading a dozen of them teaches the pattern of what frontier-scale AI serving does not catch on its own; those are exactly the patterns the AI SRE's runbook library and fixture back-feed (chapter 05) exist to prevent.

This chapter codifies the dissection method, enumerates the corpus the AI SRE should read, and teaches the synthesis step — extracting the pattern across incidents so that the role's own detection, response, and prevention surface improves.

## 1. What a dissected public post-mortem produces

The deliverable of dissecting a public post-mortem is not prose. It is a structured record that composes with the internal corpus: the same post-mortem fields from chapter 03 §2, filled in from the external document, with explicit links back to the fields the original writer did not include. The record is useful because it:

- **Classifies the incident in the AI SRE function's taxonomy** (chapter 03 §3). A vendor's post-mortem does not have to use the same primary-cause vocabulary; the dissection re-indexes it so the internal corpus is searchable.
- **Named signal, named mitigation, named prevention.** The public write-up usually names at least two of these; the dissection notes the gaps.
- **Transfers the fixture candidates** to the internal library. "They added a per-replica rollout gate" becomes "we should have an equivalent gate; do we?"
- **Benchmarks our response envelope.** A public TTD / TTR number sets a credible target against which the internal numbers can be judged.
- **Feeds chapter 05's back-feed cycle.** If a public incident is of a class the internal fleet has not had, the fixture back-feed does not wait for the internal recurrence.

A dissection record is short (one page) and template-bound (section 3 below). The point is comparability across the corpus.

## 2. Where the public corpus lives

The sources below are where the AI SRE function regularly reads. The specific URLs drift; the publishers do not.

### 2.1 Frontier-lab engineering blogs and incident disclosures

- **OpenAI.** The company maintains a public status page (`status.openai.com`) with incident history and publishes engineering write-ups on its research / blog surfaces for larger events. The 11 December 2024 ChatGPT outage is one such publicly post-mortem-ed event: the status-page incident record ([status.openai.com/incidents/ctrsv3lwd797](https://status.openai.com/incidents/ctrsv3lwd797)) describes the root cause as a newly deployed telemetry service that overloaded the Kubernetes control plane and broke DNS-based service discovery across every cluster.
- **Anthropic.** Status page at `status.anthropic.com`; incident write-ups published on the engineering blog when a customer-visible event warrants — e.g. ["A postmortem of three recent issues"](https://www.anthropic.com/engineering/a-postmortem-of-three-recent-issues) (September 2025), which dissected the August–September 2025 Claude quality-degradation incidents across a context-window routing bug, an output-corruption bug, and an XLA:TPU compiler latent bug.
- **Google DeepMind / Google Cloud Vertex AI.** Google Cloud publishes incident reports through the Personalized Service Health interface and the Google Cloud status page (`status.cloud.google.com`); Vertex AI and Gemini API incidents appear there with per-incident records (e.g. the December 2024 multi-region 5xx surge on the Gemini API and the February 2026 global-endpoint error-rate incident). Detailed engineering retrospectives beyond the status-page record are less consistently published.
- **Microsoft Azure OpenAI / Azure AI Foundry.** Azure Service Health and the Azure status history (`azure.status.microsoft`) publish Post-Incident Reviews (PIRs) per tracking ID for impacted subscriptions — typically within 5–10 business days of mitigation — covering Azure OpenAI Service, Foundry Agent Service, and Foundry Models.
- **AWS Bedrock.** Bedrock incidents are tracked on the AWS Health Dashboard, and AWS publishes Post-Event Summaries (PES) for larger regional / service events. The October 2025 `us-east-1` DynamoDB / DNS cascade — which took Bedrock offline alongside many other AWS services — is a recent example where a PES was published after the fact.

### 2.2 Cloud-infrastructure provider post-mortems for the underlying substrate

AI systems' root causes often live in the cloud provider's substrate, not the AI vendor's. The AI SRE reads these directly:

- **AWS Service Health Dashboard.** Post-event summaries for regional / service outages.
- **Google Cloud Status Dashboard and Public Issue Tracker.**
- **Azure Status History.**
- **Cloudflare blog incident write-ups.** Many AI endpoints transit Cloudflare and are correlated with its availability; the most directly AI-relevant published retrospectives are the [12 June 2025 Workers KV / Workers AI outage](https://blog.cloudflare.com/cloudflare-service-outage-june-12-2025/) (all inference requests to Workers AI failed for the duration) and the [18 November 2025 outage](https://blog.cloudflare.com/18-november-2025-outage/), which took major third-party AI chatbots offline alongside Cloudflare's own services.
- **Fastly, Akamai, and other CDN / edge providers** when relevant to the deployment.

### 2.3 Independent incident catalogues

- **AI Incident Database** (`incidentdatabase.ai`), MIT / Partnership on AI — curated real-world AI incidents including safety and quality events. Grounded in `mod-105` chapter 02.
- **OECD AI Incidents and Hazards Monitor (AIM)** ([oecd.ai/en/incidents](https://oecd.ai/en/incidents)) — the OECD's automated, cross-jurisdiction catalogue that detects AI incidents and hazards from international news outlets in near-real-time.
- **NIST AI RMF Generative AI Profile risk taxonomy** — the mapping from incident to risk class; useful for cross-indexing.
- **The MIT AI Risk Repository** — risk catalogue, grounded in `mod-105` chapter 02.
- **News-of-record reporting** — The New York Times, Reuters, Bloomberg, the Verge, 404 Media, etc. for events that a vendor has not (yet) published a formal post-mortem on but has acknowledged publicly. News reporting is a *source*, not a *post-mortem*; the dissection record is explicit about which fields are from reporting vs. from vendor publication.

### 2.4 The AI SRE's reading cadence

- **Weekly.** Status-page history for every vendor in the dependency map (`mod-107` chapter 01).
- **Monthly.** Engineering-blog digest for the frontier labs; AI Incident Database and OECD.AI updates.
- **Per-event.** When a public incident of a class the fleet depends on is reported, the function assigns a dissection within 48 hours.
- **Quarterly.** A synthesis across the quarter's dissections — what patterns are rising, what fixture categories the corpus suggests we should prioritise.

## 3. The dissection template

The dissection record mirrors the internal post-mortem template (chapter 03 §2) with explicit "unknown — external" entries where the public document does not provide the field.

```markdown
# Dissection: <vendor> <incident title>

## Source

- URL: <permanent link>
- Publisher: <vendor | cloud provider | news outlet | catalogue>
- Publication date: <date>
- Our reading date: <date>

## Summary (internal paraphrase, 2–3 sentences)

<what happened, scope, mitigation, outcome, in our vocabulary>

## Classification

- Primary cause category (chapter 03 §3): <model-weight | kernel | scheduler | prompt-config | retrieval | provider | agent-cascade | eval-judge-drift>
- Matching internal runbook (chapter 01 §3): <rb-...>
- Mapped mod-105 failure class: <class>

## Timeline (from the source; UTC)

<flat chronological list; mark inferences explicitly>

## Impact

- User-visible: <paraphrase>
- Duration: <from vendor; ours if dependent>
- Scope: <routes, geographies, product surfaces>

## What the vendor said went wrong

<quoted or paraphrased technical narrative>

## What the vendor said they fixed / are fixing

<quoted or paraphrased remediation>

## Our translation

- What signal would have caught this on our stack?
- Do we have the signal? If not, is it in our fixture-backlog?
- What mitigation would we have run?
- Is our matching runbook (above) sufficient? If not, what addition does this incident argue for?

## Candidate fixture back-feeds for us

<list, each as a fixture entry per chapter 03 §2.9>

## Gaps in the public document

<fields we would want but the publisher did not include>
```

A dissection is not an op-ed about the vendor. It is a practice document. The author's job is to extract value for the function's own operation, not to score the vendor.

## 4. The patterns that recur

Reading a dozen dissections across the frontier labs yields a short list of patterns that appear repeatedly. The AI SRE's library-authoring focus is sharpened by them.

### 4.1 Observability catches the wrong layer first

A frequent shape: an incident whose root cause is in the AI serving layer or the model itself surfaces initially as a classical-looking symptom in the substrate (CPU exhaustion in a control-plane pod, a disk-full alert, a network incident) and the real cause is diagnosed only later. The December 2024 ChatGPT outage's published narrative is an example of this shape — the proximate symptoms were Kubernetes-control-plane and DNS-resolution failures, and the upstream cause (a newly deployed telemetry service that generated overwhelming Kubernetes API load) was identified only after responders regained control-plane access ([status.openai.com/incidents/ctrsv3lwd797](https://status.openai.com/incidents/ctrsv3lwd797)).

The AI SRE's takeaway: AI-adjacent observability (`mod-104` chapter 06) must not be an afterthought added to a substrate-focused dashboard. The dashboards that include the AI-specific signals tend to compress diagnosis time dramatically.

### 4.2 Progressive rollout is where most model-weight incidents turn

A large fraction of model-weight incidents in the public record are caught (or not caught) by the progressive-rollout tooling. The lab publishes "we rolled out to X% of traffic, the eval canary caught it at Y%, we rolled back in Z minutes" and the number Z is the headline metric. When the lab does not catch it in progressive rollout — often because the canary eval is not sensitive to the quality regression at hand — the incident is much longer.

The AI SRE's takeaway: the progressive-rollout surface and its canary eval are the single most important preventive control for model-weight incidents. `mod-111`'s release-engineering module is where this lives; this chapter's dissection practice feeds that module.

### 4.3 Vendor transparency correlates with recovery

Reading across incidents, the vendors who publish a specific technical write-up within days of resolution tend to have the shortest follow-on incident windows. The reasons are system-shaped: a write-up implies a complete internal retrospective; a complete retrospective implies fixtures land; fixtures imply the next event is a different event. The absence of a public retrospective is not evidence of absence of learning, but the correlation holds often enough that the AI SRE function treats "does the vendor publish detailed post-mortems" as a vendor-eval signal (`mod-107` chapter 06).

### 4.4 Safety incidents behave differently from availability incidents

When a safety-adjacent incident becomes public — a model emits unacceptable content, a jailbreak surface is reported, a demonstration output is incorrect — the pattern of response and publication is different. The vendor frequently publishes a policy / safety-team narrative rather than an engineering post-mortem; the engineering specifics are often not in the public record. Google's February 2023 Bard demo, in which Bard's publicly demonstrated response contained a factual error about the James Webb Space Telescope, is an example of a safety-adjacent event that was extensively discussed in reporting (and cost Alphabet roughly $100B of market capitalisation in one day) but was not accompanied by a formal public engineering retrospective from Google.

The AI SRE's takeaway: safety incidents cross into legal / PR / policy early and public engineering retrospectives on them are rarer. When dissecting them, the fields the public document does not have (specific detection signal, specific internal mitigation) are the ones the function invests in internally.

### 4.5 Jailbreaks and misuse often look like reliability problems in production

Published jailbreak and misuse events — the Microsoft Bing "Sydney" period in early 2023, documented most prominently in [Kevin Roose's 16 February 2023 New York Times transcript and column](https://www.nytimes.com/2023/02/16/technology/bing-chatbot-microsoft-chatgpt.html) and in subsequent reporting, is a frequently referenced example — surface as a mix of safety and reliability behaviour: users reliably elicit outputs the vendor did not intend, and the vendor patches (Microsoft responded with chat-length caps and system-prompt changes rather than an engineering post-mortem), but the pattern of patching is incident-shaped.

The AI SRE's takeaway: safety-incident patterns inform the detection surface the AI SRE owns with the safety function. A spike in off-distribution prompts is a reliability signal as much as it is a safety signal, and the fixture back-feed often lives in the same `mod-105` chapter 04 detection surface.

### 4.6 Vendor outages are correlated

The frontier labs, cloud providers, and CDNs share substrate. A Cloudflare incident often correlates with AI endpoint availability events across multiple vendors. A GCP incident impacts Vertex AI and anyone whose backbone traverses GCP. Reading cloud-provider post-mortems alongside AI-vendor post-mortems reveals which events the fleet experienced as a single incident and which were correlated-but-separate. `mod-107` chapter 05's "shared substrate" correction to the SLA composition math is the direct consequence.

### 4.7 The dependency-depth is often where things go wrong

Many public incidents trace to a dependency the vendor itself did not list as critical: an auxiliary service, a rarely-used provisioning path, a telemetry component that becomes load-bearing under incident conditions. The AI SRE's dependency map (`mod-107` chapter 01) and this function's chaos program (`mod-109`) are the practices that prevent this at the AI SRE's layer; the dissection practice is where the pattern is first noticed.

## 5. The synthesis step

A single dissection informs a single runbook amendment. The synthesis step takes a quarter of dissections and asks four questions:

- **Which failure classes recurred?** Three model-weight incidents across the quarter's reading suggests the internal prevention surface should prioritise model-weight (progressive-rollout canary, version-pinning, checksum verification).
- **Which signals detected first?** If telemetry-shaped signals caught most of the incidents the public record disclosed, the internal AI-serving dashboards should include equivalent telemetry.
- **Which mitigations held?** The successful mitigations across the corpus are the ones the internal fallback contract should codify.
- **Which gaps are repeated?** When multiple vendors' public write-ups show the same gap ("we lacked a per-replica rollout gate"), the internal fixture back-feed is the place to not have that gap.

The synthesis is published inside the AI SRE function quarterly. Exercise 04 asks the learner to run the method end-to-end on a curated set of public incidents.

## 6. Reading responsibly

Three ethical points about reading and citing public post-mortems:

- **Do not re-attribute.** If the vendor's public document does not name an engineer, the AI SRE's dissection does not either.
- **Do not speculate publicly.** The dissection record is an internal artifact. Public commentary on another vendor's incident is a PR / legal / policy decision, not an engineering decision.
- **Respect user data if cited.** If the incident involved exfiltration or leakage of user interactions, the dissection may need to redact sample content to the same degree the vendor did. When in doubt, legal reviews.

## 7. The AI SRE's dissection worksheet

1. **Pick an incident** from the weekly or monthly feed.
2. **Open the template** in §3.
3. **Fill the source, summary, and classification** sections from the publisher's narrative.
4. **Walk the timeline** from the publisher's facts; mark your inferences explicitly.
5. **Translate.** Would our signal have caught this? Would our runbook have mitigated it? Would our fallback contract have absorbed it?
6. **Name candidate fixtures.** Chapter 05's back-feed cycle consumes these directly.
7. **Note the gaps in the public document.** These become our monitoring plan for the vendor's next event.
8. **File the dissection** in the internal corpus, tagged by primary-cause category and runbook.
9. **Contribute to the quarterly synthesis.**

## Summary

- **The public corpus is a library.** The frontier labs, the cloud providers, the AI incident catalogues, and independent reporting are the dossier an AI SRE consults regularly.
- **Dissect, don't editorialise.** The dissection record uses the internal post-mortem template, with explicit markers for fields the public document does not supply.
- **Patterns recur.** Observability catching the wrong layer first, progressive-rollout as the dominant preventive control, vendor transparency correlating with recovery, safety incidents behaving differently, jailbreaks surfacing as reliability problems, shared-substrate correlation, and dependency-depth surprises are the most common.
- **Synthesis closes the loop.** A quarterly roll-up translates the external pattern into internal fixture priorities and runbook amendments.
- **Read responsibly.** Do not re-attribute, do not speculate publicly, and redact where the vendor did.

Chapter 05 formalises the back-feed cycle that this chapter and chapter 03 both terminate in: every post-mortem, internal or dissected, produces at least one permanent fixture that closes the SLO stack's loop.
