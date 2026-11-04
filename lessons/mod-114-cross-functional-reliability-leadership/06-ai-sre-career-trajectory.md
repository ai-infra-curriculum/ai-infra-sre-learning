# Positioning the AI-SRE Career Trajectory: Levels, Certifications, Open-Source Contribution

## Why this chapter exists

Mod-101 placed the AI SRE role at level 35 on this org's infrastructure ladder — a peer of the inference-engineer, performance, and security peers; above the ML-platform and eval peers; above mlops; above the consumer tracks. A reader who has worked through the preceding fourteen modules is now operating at that level and is asking the next question: *where does this lead?* What does senior engineer, staff, architect, principal look like for someone whose craft is AI-specific reliability? What portfolio of experience, credentials, and public contribution carries weight when the market reads for the role in 2026 and beyond?

This chapter answers at team-leadership altitude. It is written honestly — the AI SRE role is new enough (mod-101 chapter 06) that none of the senior tiers have canonical shapes yet; frontier labs, hyperscalers, enterprise AI platforms, and regulated-industry AI deployments each pull the senior shape in different directions. The chapter writes what is stable: the four senior tiers' operating altitude, the load-bearing certifications that enterprise hiring reads, the open-source toolchain surfaces where a function engineer compounds career capital, and the anti-patterns that stall the trajectory.

Primary references: mod-101 chapter 01 and chapter 06 for the ladder and the hiring-market frame this chapter extends; the [CNCF certification catalogue](https://www.cncf.io/training/certification/) for the Kubernetes credentials; the [NVIDIA Deep Learning Institute](https://www.nvidia.com/en-us/training/) and the [(ISC)² CISSP](https://www.isc2.org/Certifications/CISSP) credential for the enterprise-hiring-market signal.

## 1. The four altitudes above level 35

Each altitude is defined less by title (titles vary — "senior," "staff," "principal," "distinguished," "fellow" — across organisations) and more by the scope of the work and the time-horizon of the decisions.

### 1.1 Senior AI SRE (approximately level 40 adjacent)

The senior tier owns a significant reliability surface end-to-end: an entire tier-1 service's reliability contract, a critical cross-cutting capability (the observability platform, the chaos programme, the release-gate service), or a major migration (the chapter 02 RFC's cutover). The scope is **larger than one chapter of this curriculum** but still bounded by the function's charter.

- **Decision horizon.** One to two quarters forward.
- **Signature artifacts.** A signed reliability-engineering RFC of the chapter 02 shape; a signed error-budget policy (mod-103 chapter 04); a runbook library at the quality level of mod-108 chapter 01; multi-service SLO composition work (chapter 03 §9).
- **Composition with peers.** The senior tier is the function's day-to-day interface with level-35 peer tech leads. The peer-interface contracts (chapter 03) are drafted, reviewed, and renegotiated at this level.
- **Mentorship.** The senior tier mentors mid-level engineers through their first on-call rotations, their first post-mortems, their first drill facilitations. Mod-108 chapter 03's shadow-oncall convention is run by the senior tier.

The transition to senior is about demonstrating durable ownership of a reliability surface across an incident, a migration, or a post-mortem's full action-item cycle. A single incident handled well does not promote; a quarter of a service's reliability posture improving against its SLO does.

### 1.2 Staff AI SRE (approximately level 42–45 adjacent)

The staff tier owns the function's **cross-service posture** or a **cross-team capability** that multiple SRE teams depend on. This tier is the function's second-order leverage: the person who makes two teams of SREs more effective rather than one team's engineers more effective. In organisations without a distinct `ai-infra-architect` role at level 40, the staff tier may fulfil parts of it.

- **Decision horizon.** Two to four quarters forward.
- **Signature artifacts.** The reliability-engineering RFC authored across multiple services; the team's contribution to the organisation's multi-year reliability roadmap; the function's leadership artifact when a major cross-team incident occurs (the cross-functional post-mortem mod-108 chapter 04 §4).
- **Composition with peers.** The staff tier interfaces with peer *tech leads* across inference-engineer, performance, security, ml-platform, eval, and mlops. The quarterly peer-interface health check (chapter 03 §10) is run at this level; disputes that cannot be resolved by team leads escalate here before reaching the architect.
- **Mentorship.** The staff tier mentors senior engineers through their first RFCs, their first cross-team migrations, and their first upward-communication quarterly narratives.

The transition to staff is about demonstrating that the function's posture improved under your influence across services you did not individually operate. Evidence: the composition artifact (chapter 03 §8) is in Git, is current, and is referenced across the organisation. The maturity-ladder score (chapter 01 §5) advanced.

### 1.3 Architect AI SRE (approximately level 45–48 adjacent, aligned with `ai-infra-architect`)

The architect tier owns the function's **composition with the wider platform** at team-of-teams scope. This is the level at which mod-113 chapter 07 §2 named the `ai-infra-architect` peer — but for the AI SRE function specifically, with explicit multi-service reliability expertise. The tier may be a dedicated role or a staff engineer with an architect's charter.

- **Decision horizon.** Four to eight quarters forward.
- **Signature artifacts.** The multi-team reliability architecture document; the organisation's reliability-tooling platform strategy; the composition reference the per-service RFCs (chapter 02) build into.
- **Composition with peers.** The architect tier interfaces with the engineering leadership and with the `ai-infra-senior-architect` peer (mod-113 chapter 07 §3). Chapter 05 §3's architect-escalation ticket is received at this tier; chapter 05 §4's board-adjacent narrative is authored jointly with the director at this tier.
- **The governance boundary.** The architect does not drift into the `head-of-ai-governance` or CAIO scope. Mod-110 chapter 06 and chapter 05 §4.4 of this module are the explicit boundaries.

The transition to architect is about the function's influence beyond its own team scope: the team-of-teams DR game-day (mod-113 chapter 05 §1.3), the shared observability platform strategy (chapter 02 of this module at team-of-teams scope), the reference reliability architecture that successive teams build inside.

### 1.4 Principal / Distinguished AI SRE (approximately level 48–55 adjacent)

The principal / distinguished tier operates at the horizon where the function's **shape itself** is being redesigned. This is the level at which the organisation's AI-reliability charter is written and rewritten; where the senior-architect peer (mod-113 chapter 07 §3) is a peer; where external posture — board-adjacent presence, upstream open-source influence, regulator-facing technical representation — becomes part of the role.

- **Decision horizon.** Two to five years forward.
- **Signature artifacts.** The organisation's AI-reliability charter; multi-year roadmap; upstream open-source leadership (§5 below); external technical representation (conference keynotes, standards participation, regulator engagement).
- **Composition with peers.** The principal tier interfaces with the C-suite, the senior-architect peers across adjacent organisations, and the industry at large. Within the organisation the tier is a technical-strategy voice; outside, it is the organisation's visible AI-SRE expertise.

The path to principal is not described well by a single checklist. The pattern is durable: multi-year impact on multiple reliability programmes, visible contribution to the discipline's public toolchain or standards, successful mentorship of successor leaders at the staff and architect tiers. The transition is often from the architect tier via years of visible cross-industry influence.

## 2. The honest caveat on title mapping

The four altitudes above describe scope, decision horizon, and signature artifacts; the **titles** vary. Frontier labs often use "senior → staff → principal" with fewer tiers; hyperscalers often use "L5 → L6 → L7 → L8" numeric tiers that span ten levels on the ladder; enterprise shops often use "principal engineer → distinguished engineer → fellow." The function reads any specific posting against the scope description, not against the title, and the function's self-positioning in its chapter 05 narratives does the same.

This caveat is doubly true for the AI SRE role as of 2026. Mod-101 chapter 06 documented the fragmentation: the title "AI SRE" / "LLM SRE" / "AI Production Engineer" has emerged but is still being disambiguated from adjacent roles. The function reads a specific posting by matching the stated scope against the mod-101 chapter 02 peer deferral map; the chapter 05 upward narrative does the same for self-positioning. <!-- needs-research: a public 2026 posting sample across the enterprise, hyperscaler, and frontier-lab segments showing the scope / title correspondence, to be backfilled when the posting-corpus research sampling (flagged in mod-101 resources.md) completes. -->

## 3. Load-bearing certifications in enterprise hiring

Certifications matter differently in different market segments. Frontier labs and the strongest hyperscaler-AI teams weight the certification signal lightly — public project history and interview-measured depth dominate. Enterprise-AI deployments and regulated-industry AI platforms (healthcare, financial services, public sector, defence) weight certifications more heavily — they map onto specific compliance frameworks and specific hiring-criterion checklists. The certifications below are the ones that land on enterprise job-description "required" or "preferred" lists with reliable frequency; the function chooses the ones that match the target market segment.

### 3.1 Kubernetes-platform credentials

- **[CKA — Certified Kubernetes Administrator](https://www.cncf.io/training/certification/cka/).** The operator-focused Kubernetes certification. Hands-on, time-boxed, exam-based. For an AI SRE whose production fleet runs on Kubernetes (which is the overwhelmingly common case as of 2026), CKA is the single most frequently-requested credential on enterprise job descriptions.
- **[CKAD — Certified Kubernetes Application Developer](https://www.cncf.io/training/certification/ckad/).** The application-focused Kubernetes certification. For an AI SRE who co-authors CRDs (mod-112 chapter 03), Helm charts, and operators, CKAD complements CKA. If forced to choose one for the first year of a role, CKA is the broader signal; if the role is heavy on in-cluster application plumbing, CKAD is the stronger match.
- **[CKS — Certified Kubernetes Security Specialist](https://www.cncf.io/training/certification/cks/).** Requires CKA as prerequisite. For roles where the function interfaces heavily with the `ai-infra-security` peer (mod-110) or where the production environment is regulated (financial services, public sector), CKS is a differentiator.

### 3.2 GPU-platform credentials

- **[NVIDIA DLI (Deep Learning Institute)](https://www.nvidia.com/en-us/training/) courses and certifications.** Specific courses load-bearing for an AI SRE:
  - *Fundamentals of Deep Learning for Multi-GPUs* — the GPU-platform operator-level credential.
  - *Accelerating CUDA C++ Applications with Multiple GPUs* — only if the function expects to spend time adjacent to the `ai-infra-performance` peer's craft; the credential is less common on SRE postings and more common on performance postings.
  - *Fundamentals of Accelerated Computing with CUDA Python / C++* — foundational; mandatory if the function does any kernel-adjacent work.
  - *Building LLM Applications with Prompt Engineering* and related LLM-platform courses — these overlap with the `llm-application-developer` consumer track (mod-101 chapter 02) and are useful for peer-fluency rather than for the SRE's own craft.
- **NVIDIA-Certified Associate / Professional credentials.** More enterprise-facing than DLI; useful for roles where the organisation values vendor-recognised certification signal.

### 3.3 Security and BCP credentials

- **[(ISC)² CISSP (Certified Information Systems Security Professional)](https://www.isc2.org/Certifications/CISSP).** The credential that carries the most weight for an AI SRE in a regulated or enterprise environment, specifically around BCP, incident-response, and security-reliability crossover credibility (mod-110, mod-113). The CISSP's Common Body of Knowledge includes security-operations and BCP domains that compose directly with this curriculum's mod-108, mod-110, and mod-113. Requires five years of qualifying work experience plus the exam; the experience bar is the gate most candidates underestimate.
- **[CISM (Certified Information Security Manager)](https://www.isaca.org/credentialing/cism)** — less common than CISSP for SRE roles but shows up in management-adjacent AI-infra-leadership postings.
- **[ISO 22301 Lead Implementer](https://www.iso.org/standard/75106.html)** — niche but strong signal in regulated-industry roles where the organisation's BCP programme is being formally certified. Composes with mod-113 chapter 06's artifact pack.

### 3.4 Cloud-provider reliability-specialty credentials

Each major cloud provider publishes one or more reliability- or operations-adjacent specialty credentials. The function reads the posting's target cloud and selects the matching credential.

- **AWS.** [AWS Certified SysOps Administrator (Associate)](https://aws.amazon.com/certification/certified-sysops-admin-associate/); [AWS Certified DevOps Engineer (Professional)](https://aws.amazon.com/certification/certified-devops-engineer-professional/); the [AWS Well-Architected Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/) is the substrate the exams draw from. Historically AWS also offered a specialty-level operations credential; the specific credential names and the roster evolve — the function checks the current roster at exam-booking time.
- **Azure.** [Microsoft Certified: Azure Administrator Associate (AZ-104)](https://learn.microsoft.com/en-us/certifications/azure-administrator/); [Microsoft Certified: DevOps Engineer Expert (AZ-400)](https://learn.microsoft.com/en-us/certifications/devops-engineer/); the [Azure Well-Architected Reliability pillar](https://learn.microsoft.com/en-us/azure/well-architected/reliability/) is the substrate.
- **Google Cloud.** [Google Cloud Professional Cloud DevOps Engineer](https://cloud.google.com/certification/cloud-devops-engineer); [Professional Cloud Architect](https://cloud.google.com/certification/cloud-architect); the [Google Cloud Architecture Framework reliability pillar](https://cloud.google.com/architecture/framework/reliability) is the substrate.

<!-- needs-research: the specific current (2026) roster of cloud-provider "reliability" or "operations" specialty credentials, including whether AWS's recently-rotated specialty line-up includes a direct AI / ML operations credential. The vendors rotate the roster faster than most other elements of the enterprise hiring signal; cite the current rate card rather than a frozen list. -->

### 3.5 Picking the certification mix

The function does not pursue all of these in parallel. A rough priority for a mid-career (level 35) AI SRE looking at the next three years:

1. **CKA first.** Highest hit-rate on enterprise job descriptions; the Kubernetes-fluency signal is foundational.
2. **One cloud-provider operations-level credential** matching the target market (AWS SysOps / AZ-104 / Google Professional Cloud DevOps Engineer).
3. **NVIDIA DLI foundational course** — the GPU-platform operator credential; signals to the market that the function has moved from classical SRE to AI SRE in a verifiable way.
4. **CISSP** when the role or target role is in regulated or enterprise-heavy environments and the five-year experience gate is met.

A function in the frontier-lab segment may invest none of this time in certifications and instead invest in the §5 open-source contribution surface. The segments are real and the choice is explicit.

## 4. The portfolio: evidence beyond certifications

Certifications signal baseline capability. Portfolio signals signal level. For an AI SRE moving from level 35 to staff / architect / principal, the portfolio evidence that carries weight:

- **Signed reliability-engineering RFCs.** Chapter 02-shape documents authored and shipped, with the migration completed and the post-migration review (chapter 02 §5.5) in Git. A function at senior tier has one; at staff tier has two to three; at architect tier has authored the reference architecture the RFCs build inside.
- **Published post-mortems.** The function's own name on a post-mortem that has been shared externally — on a company engineering blog, in a conference talk, or (less commonly) in a regulator-facing filing. The post-mortem's quality (the mod-108 chapter 04 structure, blameless framing, honest causal attribution) is what gets read.
- **Open-source contribution history.** §5 below.
- **Conference participation.** Talks at SREcon, KubeCon, or domain-specific AI-reliability events; workshops run; panel participation. Staff tier may have one or two talks per year; architect and principal tiers have ongoing visibility.
- **Mentorship ledger.** A written record of the engineers mentored, the specific scopes (first on-call, first RFC, first post-mortem, first upward brief), and the outcomes. Mentorship is load-bearing for the staff → architect → principal transitions and is often under-documented until the promotion case needs the evidence.
- **Mobility and breadth.** For the architect and principal tiers, having operated the function in more than one organisation — ideally in more than one market segment (hyperscaler + enterprise, or frontier-lab + regulated-industry) — is a signal the market reads positively. The function that spent fifteen years in one organisation has deep expertise but may lack cross-segment pattern recognition.

## 5. Contributing to the upstream open-source reliability toolchain

Open-source contribution is the career surface that compounds. A function engineer who ships one well-landed contribution per quarter over five years has a public record the market reads directly; a function engineer whose entire portfolio is internal has to re-establish credibility at every job change.

The AI-reliability toolchain has four substrate areas where a function engineer's day-job expertise translates cleanly into upstream contribution opportunities.

### 5.1 OpenTelemetry

The [OpenTelemetry](https://opentelemetry.io/docs/) project is the function's instrumentation substrate (mod-104). Contribution surfaces:

- **The [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/).** The specification of standard attribute names for AI / LLM telemetry. The specification is actively evolving as of 2026; the function's day-job experience with per-provider, per-model-version, per-tool-call telemetry is directly relevant. Contributions include proposed attributes, clarifications on existing attributes, and language-SDK implementations.
- **The [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector) and [Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib) repositories.** Receivers, processors, exporters, connectors. A function running a collector in production has a direct feedback loop into improvements — a bugfix for the Prometheus receiver under high-cardinality AI telemetry, a new processor for GenAI-specific attribute transformation, an exporter for a specific backend.
- **Language-SDK instrumentation.** Specific AI library instrumentations (for inference engines, orchestration libraries, agent frameworks) are often first-party or contributed. A production user of a specific stack has the context to contribute the instrumentation.

Where to start: the [OpenTelemetry Contributing guide](https://opentelemetry.io/community/) and the SIG-GenAI working-group channels.

### 5.2 Prometheus ecosystem and the GPU-telemetry surface

- **[DCGM-exporter](https://github.com/NVIDIA/dcgm-exporter).** NVIDIA's open-source Prometheus exporter for GPU telemetry. The function's production experience with the exporter (mod-104 chapter 04) surfaces into contributions: new metric exposures, bug fixes under specific driver / Kubernetes versions, documentation improvements for the GPU-operator composition (mod-104 chapter 04).
- **Custom Prometheus exporters.** The function often writes an internal exporter for a specific AI-infrastructure signal (inference-engine-specific metrics, retrieval-index health, prompt-registry activity). Promoting the exporter to upstream status — as a new project in the Prometheus community, or as a contribution to an existing exporter — is a classic function-engineer contribution path.
- **[Prometheus](https://github.com/prometheus/prometheus) itself, [Alertmanager](https://github.com/prometheus/alertmanager), [Pushgateway](https://github.com/prometheus/pushgateway).** Less commonly a surface for function-engineer contribution (the core is heavily maintained by dedicated teams), but documentation improvements, recording-rule patterns in the community, and PromQL community support are open.

### 5.3 Chaos engineering

- **[Chaos Mesh](https://chaos-mesh.org/docs/).** The CNCF chaos-engineering project. Function engineers contribute: new experiment types specific to AI workloads (GPU-fault injection, prefix-cache pollution, prompt-registry corruption, provider-status-page simulation — the chapter 02 of mod-109's experiment catalogue surfaces here), bug fixes in the controllers, documentation of AI-specific experiment patterns.
- **[LitmusChaos](https://litmuschaos.io/docs/).** The alternative CNCF chaos-engineering project. Similar contribution patterns; the organisation's choice between the two often defaults on which one the team already runs.
- **[kube-monkey](https://github.com/asobti/kube-monkey)** and other simpler chaos tools — smaller surfaces, lower activity, occasionally relevant.

### 5.4 Progressive delivery and release

- **[Flagger](https://github.com/fluxcd/flagger)** (part of Flux CD). Progressive-delivery controller for Kubernetes. Function engineers contribute: new metric-provider integrations (for AI-specific release-gate signals from mod-111 chapter 05), webhook extensions, canary-analysis strategies, documentation.
- **[Argo Rollouts](https://github.com/argoproj/argo-rollouts).** Argo's progressive-delivery controller. Similar contribution surface.
- **[OpenFeature](https://openfeature.dev/)** — CNCF feature-flagging standard. Contributions relevant for function engineers interfacing heavily with mod-112's config-pipeline surface.

### 5.5 How to land a first contribution

The function engineer's first upstream contribution usually fails one of four ways:

- **The "drive-by patch."** A fix for a problem the function hit in production, submitted without engaging the project's community, without the project's tests passing. Rejected or quietly deferred. Fix: engage before you patch — comment on an existing issue, join the project's Slack / matrix / mailing list, offer to work on a scoped issue the maintainers have labelled "good first issue."
- **The "over-scoped RFC."** A redesign proposal made before the function has demonstrated competence on small changes. The project's maintainers do not know the contributor; the proposal is correctly rejected. Fix: build visible commit history on small changes first.
- **The "silent abandonment."** A PR opened and then not responded to when reviewers comment. The PR goes stale; the function loses the window. Fix: treat open PRs with the same discipline as production incidents — a response SLA.
- **The "internal-only patch."** The function solves a problem internally and never offers the fix upstream. Over years the internal patch stack grows; the function's engineers become specialists in the internal fork; mobility declines. Fix: a rule that any internal-only patch to an open-source dependency is written with the intent of upstreaming within one quarter.

## 6. Composing the career plan with the day job

Chapter 01's programme rhythm leaves slack. The function engineer uses the slack explicitly:

- **Weekly.** One to two hours on the current certification study or on the current open-source issue. Not more — the day job's weekly rituals (chapter 01 §2.1 and §2.3) and on-call commitment take priority.
- **Monthly.** One upstream PR merged or one cert-exam chapter completed. The monthly SLO review (chapter 04) is where the function engineer's attention is — career work fits around it.
- **Quarterly.** One signed RFC co-authored or one conference-talk CFP submitted. The quarterly retrospective (chapter 01 §2.4) is where the function reviews the quarter's career progress alongside the operational progress.
- **Annually.** One certification earned or one visible public artifact (talk, major open-source contribution, external blog post). The annual programme review (chapter 01 §2.5) includes a career-progress reflection per team member.

The team lead's responsibility (chapter 01 §2.4) includes helping each team member articulate their career plan at the quarterly retrospective. A function whose engineers do not have career plans is a function whose engineers leave at a higher-than-necessary rate.

## 7. The anti-patterns that stall the trajectory

- **The certification stack without the portfolio.** Five certifications earned, no RFCs authored, no visible contribution history. The market reads certifications as baseline, not level. The function cannot promote past senior on certifications alone.
- **The portfolio without the depth.** One well-known blog post repeated across conferences for three years. The pattern-recognition window is open for one year; by year three the market has read the post, and the function needs the next contribution.
- **The senior-tier refusal to mentor.** A senior engineer who refuses the mentor load does not move to staff. Staff is defined in part by second-order leverage; refusing mentorship is refusing the leverage.
- **The architect-tier governance drift.** An architect engineer who starts writing governance narratives (chapter 05 §4.4 anti-pattern) drifts out of their own function. The architect peer whose reliability authority declines does not become principal.
- **The frontier-lab-only posture.** A function engineer who only reads frontier-lab practice and ignores enterprise and regulated-industry patterns has a narrow mobility window. The staff → architect transition benefits from exposure to multiple market segments.
- **The enterprise-only posture.** The symmetric anti-pattern. An engineer who only operates in heavily-regulated enterprise environments and does not track frontier practice is working from a frozen vocabulary; the discipline evolved past them.

## 8. The honest closing note

The AI SRE role is new. The four altitudes above are the stable ones; the specific titles are not. A reader who is deciding whether to invest three to five years in this trajectory is making a bet on the role continuing to clarify through 2026–2030, which the hiring-market evidence (mod-101 chapter 06) suggests is a reasonable bet but not a settled one.

The hedges:

- The classical SRE discipline is mature and transfers back if the AI-specific demand recedes.
- The GPU-platform and ML-platform expertise transfers into ML-platform engineering, mlops leadership, and infrastructure-engineering generalist roles.
- The security-reliability crossover expertise (mod-110) transfers into AI-security-leadership and risk-engineering roles.
- The programme-leadership expertise from this module transfers into platform-engineering leadership and engineering-management roles.

A function engineer's trajectory is less a straight line than a progressively-widening fan of options. Chapter 06 writes the fan as it currently exists in late 2026; the reader revisits the fan annually against their own evidence.

## Summary

- Four stable altitudes above level 35 — senior, staff, architect, principal — are defined by scope, decision horizon, and signature artifacts rather than by title. The function reads any specific posting against the scope description, not the title.
- Load-bearing enterprise certifications are the Kubernetes track (CKA primarily, CKAD and CKS secondarily), the NVIDIA DLI GPU-platform courses, (ISC)² CISSP for BCP / incident-response credibility, and one cloud-provider operations-level credential matching the target market. The function picks the mix against the target market segment and the career horizon.
- Portfolio evidence beyond certifications — signed RFCs, published post-mortems, open-source contributions, conference participation, mentorship ledger, cross-segment mobility — is what signals level rather than baseline.
- The upstream open-source reliability toolchain surfaces that compose with the AI SRE day job are OpenTelemetry (GenAI semantic conventions, Collector, SDKs), the Prometheus ecosystem (DCGM-exporter, custom exporters), Chaos Mesh / LitmusChaos, and progressive-delivery controllers (Flagger, Argo Rollouts).
- The career plan composes with the day job: weekly slack for study or upstream work, monthly incremental progress, quarterly commitments, annual visible artifacts. The team lead reviews career progress at the quarterly retrospective.
- Six anti-patterns most frequently stall the trajectory: certifications without portfolio, portfolio without depth, senior-tier refusal to mentor, architect-tier governance drift, frontier-lab-only posture, enterprise-only posture.
- The AI SRE role is new; the trajectory is a bet on the role continuing to clarify. The hedges — classical SRE, ML-platform, AI-security, platform-engineering leadership — mean the invested expertise transfers even if the specific title does not persist.

This chapter closes the module. The reader who has absorbed chapters 01–06, worked through the exercises, and participated in the programme rituals has the AI SRE's team-scope reliability-leadership capability as a practice, a programme, and a trajectory — ready to operate as the function's team lead and, over the next several years, to move up the altitudes above.
