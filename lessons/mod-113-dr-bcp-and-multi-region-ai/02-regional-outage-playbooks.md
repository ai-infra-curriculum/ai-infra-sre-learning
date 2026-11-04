# Regional Outage Playbooks for AI Workloads

## Why this chapter exists

A regional outage is the DR event the function is most likely to run in anger. Full cloud-provider failures are rare; whole-region impairment — an availability-zone going red and a cascade taking the rest of the region with it, a hosted model-provider's regional endpoint returning 5xx while the global status page stays green (`mod-107` chapter 03), a bad control-plane deploy at the vendor that drops a region's routing for 20 minutes — is routine enough that every service in the charter has a nonzero chance of seeing one each year. The function must have, before the event, a playbook that names the detection, the authority to switch, the switch mechanism, the state assumptions, and the rollback. "We failover to the other region" is not a playbook. It is a sentence that costs one hour of incident time to resolve into an actual sequence of actions.

This chapter writes the sequence. It covers the four substrates the playbook touches in any AI workload: **endpoint failover** (DNS / Anycast / global load balancer), **model-weight artifact replication** (the S3-class store that holds the pinned weights), **warm-standby regions** (the capacity that absorbs the shifted traffic without a cold-start penalty), and **provider-level fallback** (when the primary failure is the inference provider itself, not the serving substrate). Each substrate is written as a per-service contract the function fills in, cites, and drills.

Primary references: [AWS Multi-Region fundamentals](https://docs.aws.amazon.com/whitepapers/latest/aws-multi-region-fundamentals/aws-multi-region-fundamentals.html), [Amazon Route 53 health checks and DNS failover](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html), [AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html), [Amazon S3 Cross-Region Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html), [Azure Traffic Manager](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview) and [Azure Front Door](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-overview), [Google Cloud External Load Balancing](https://cloud.google.com/load-balancing/docs) and [Cloud DNS routing policies](https://cloud.google.com/dns/docs/routing-policies-overview), [BGP Anycast primer](https://datatracker.ietf.org/doc/html/rfc4786), and `mod-107` chapter 02 (provider-fallback contract).

## 1. The regional-outage shape

The function begins by naming what "regional outage" can look like in practice. A playbook that optimises only for the full-loss case will under-serve the degradation cases that are far more common.

### 1.1 Full regional loss

Every endpoint in the region returns network-unreachable or is unreachable from the global load balancer. The cloud provider has declared the region degraded on its status page. Traffic in the region has nowhere to go; cross-region failover is unavoidable.

### 1.2 Partial regional impairment

A subset of availability zones is degraded, a subset of services (EC2, EKS, networking, specific API endpoints) is impaired, or a subset of model-provider endpoints in the region is returning 5xx or slow responses. The region reports "available" in some sense; the workload is not. This is the case the three-family alerting (`mod-109` chapter 05 §2) is designed to catch — the aggregate regional dashboard is green, the per-fleet SLI is red.

### 1.3 Degraded-200 regional case

The inference provider in the region is returning 2xx responses with quality collapse — more refusals, shorter completions, judge-score drop, higher error rate on parsed structured output (`mod-105` chapter 07). There is no error signal in the HTTP sense; the regional failure is a quality-signal failure. The playbook must include failover triggered by quality signals, not only by error signals.

### 1.4 Regional control-plane failure without data-plane failure

The cloud provider's control plane (the API that provisions, scales, deploys) is impaired, but the data plane (already-running pods, already-provisioned load balancers) is unaffected. The workload runs; the function cannot deploy a change, scale up, or respond to a local autoscaler event. If the incident is elsewhere, the workload is fine; if the incident requires deploys or scale, the region is effectively unusable. The playbook distinguishes these so the function does not move traffic unnecessarily.

### 1.5 The regional-outage runbook named

Each case above has a named runbook entry in the function's `mod-108` chapter 01 library:

- `rb-regional-failover` — full regional loss.
- `rb-partial-regional-impairment` — partial failure, three-family-alert-driven.
- `rb-quality-regional-degradation` — degraded-200 case.
- `rb-regional-control-plane-outage` — control-plane-only case; usually a hold-and-observe, not a failover.

Each is version-controlled, drilled per the `mod-109` cadence, and signed off on by the function's on-call lead.

## 2. Endpoint failover: DNS, Anycast, global LB

The first mechanic is moving the user's request from the failing region to the healthy region. The three available substrates — DNS-based failover, BGP Anycast, and a global load balancer — have different detection latencies, different failure semantics, and different fit for AI serving. The function picks one per workload and names why.

### 2.1 DNS failover with health checks

The classical substrate: the service's hostname resolves to a region-local endpoint via a DNS record whose TTL is low (30–60 s). The DNS provider runs health checks against each record; on failure, it withdraws the failing record and resolves to the healthy region.

- **Vendors.** [Route 53 health checks and DNS failover](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html), [Azure Traffic Manager](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview), [Cloud DNS routing policies](https://cloud.google.com/dns/docs/routing-policies-overview), [NS1](https://www.ns1.com/products/managed-dns/) / [Dyn](https://www.oracle.com/cloud/networking/dns/) as third-party options.
- **Detection latency.** Dominated by the health-check interval (often 30 s) and the TTL (60 s typical, as low as 20 s with some providers). Call it ~90 s to ~3 min on a well-tuned setup.
- **Failure semantics.** Client-side DNS caches lag the authoritative change. Even with TTL 60 s, some clients hold stale resolutions for minutes; a small tail holds for hours. The function treats DNS failover as a *statistical* cutover, not an instant one.
- **Fit for AI serving.** Acceptable for request-response inference where clients re-resolve on connection reset. Marginal for long-lived streaming connections — the client is already connected; DNS change does not disconnect it. Combine with a server-side `Connection: close` or stream-termination at a defined boundary.

### 2.2 BGP Anycast

A single IP address is announced from multiple regions via BGP; packets flow to the nearest region by routing. Withdraw the announcement from the failing region and traffic shifts within a BGP convergence window (seconds).

- **Vendors.** [AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html) and [Cloudflare's Anycast network](https://developers.cloudflare.com/network/) are the two most visible managed offerings; self-managed Anycast is operationally expensive and the function does not usually run it.
- **Detection latency.** The managed-Anycast provider's health probe runs on the order of 10–30 s. A withdrawal propagates in low tens of seconds.
- **Failure semantics.** Client cutover is network-driven, not application-driven. In-flight TCP connections on the failing region are reset; clients reconnect and hit the healthy region without a DNS refresh.
- **Fit for AI serving.** Strong fit for streaming inference. The client reconnect on withdrawal is an explicit boundary — the client knows the stream terminated and retries — rather than DNS's silent staleness.

### 2.3 Global load balancer

A single external endpoint (a cloud-provider global LB) that routes to regional backends and withdraws unhealthy backends from the pool. The client sees one endpoint; failover is invisible.

- **Vendors.** [AWS Global Accelerator with a load balancer](https://docs.aws.amazon.com/global-accelerator/latest/dg/about-endpoint-groups.html), [Azure Front Door](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-overview), [Google Cloud external HTTP(S) load balancer](https://cloud.google.com/load-balancing/docs/https). Third-party CDN / edge platforms (Cloudflare, Fastly, Akamai) offer equivalent routing.
- **Detection latency.** Backend health-check intervals in the low tens of seconds; failover near-instant from the client's perspective.
- **Failure semantics.** In-flight requests on the failing backend are either drained (if the LB supports connection draining) or reset. New requests flow to the healthy backend immediately.
- **Fit for AI serving.** Strong fit when the vendor's global LB supports streaming (HTTP/2 and SSE on AWS Global Accelerator with ALB, Azure Front Door with specific SKUs, Google Cloud HTTP(S) LB). Validate streaming support in the vendor's current documentation before assuming it; AI workloads with long-lived SSE connections hit edge cases classical web workloads do not. <!-- needs-research: verify current SSE / streaming support per vendor global-LB offering against the live documentation when selecting an endpoint substrate -->

### 2.4 The choice table

The function picks one substrate per service and records the choice:

| Substrate | Detection | Cutover | Streaming fit | Typical choice for |
|-----------|-----------|---------|---------------|---------------------|
| DNS failover | ~90 s – 3 min | Statistical (TTL-bound) | Marginal; needs server-side stream termination | Request-response REST, tier-2 workloads |
| BGP Anycast | ~10 – 30 s | Explicit (TCP reset) | Strong | Tier-1 streaming inference |
| Global LB | ~10 – 60 s | Server-initiated | Strong when vendor supports | Default on a cloud-provider-native stack |

The chosen substrate is cited in the service's ISCP (chapter 06 §3).

## 3. Cross-region replication of the model-weight artifact store

The second mechanic is ensuring the model weights the failover region needs are already present — or are reachable fast enough — when traffic shifts. Model weights are large, pinned, and immutable; they replicate well but at bandwidth cost.

### 3.1 The artifact store

The function treats model weights as artifacts in an object store. The common substrates: [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html), [Google Cloud Storage](https://cloud.google.com/storage/docs), [Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-overview). The store is content-addressed (object name includes the model hash or pinned version) and immutable per version. A new weight version is a new key; "the latest" is a pointer updated by a promotion step upstream of this chapter.

### 3.2 Cross-region replication

Each vendor's object store exposes a cross-region replication primitive:

- **S3.** [Cross-Region Replication (CRR)](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html) replicates new objects asynchronously to a destination bucket in another region. [S3 Replication Time Control (RTC)](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-time-control.html) adds a service-level objective — 99.99% of objects replicated within 15 minutes. The SLO is stated against *new* objects; a backfill of existing objects uses S3 Batch Replication.
- **Google Cloud Storage.** [Dual-region and multi-region buckets](https://cloud.google.com/storage/docs/locations) provide synchronous (dual-region) or eventual-consistency (multi-region) replication at bucket creation time; [turbo replication](https://cloud.google.com/storage/docs/turbo-replication) targets a 15-minute RTO on dual-region buckets.
- **Azure Blob Storage.** [Geo-redundant storage (GRS) and read-access geo-redundant storage (RA-GRS)](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy) replicate asynchronously to the paired region; RPO is typically under 15 minutes and is not contractual in the strict sense.

The function reads the vendor's current SLO for the chosen replication mode and cites it in the ISCP.

### 3.3 Weight propagation RTO

A 140 GB model replicated at 1 Gbps takes ~20 minutes of wire time; at 10 Gbps, ~2 minutes. Cross-region bandwidth is almost never the vendor's full fabric rate; the function measures. The **weight-propagation RTO** — the time from "weight version promoted in primary" to "weight version resolvable in secondary" — is a first-class planning metric. Common patterns:

- **Pre-staged.** Promotion pushes the weight to *both* regions before traffic sees the new version; weight-propagation RTO is driven to zero by making it a prerequisite of promotion. This is the function's default on tier-1 workloads and the pattern the release-gate (`mod-111` chapter 05) enforces.
- **Lazy replication with dwell.** The primary promotes; replication lag is measured; the canary ramps only once the secondary reports replication-complete. The dwell introduces a release-latency cost in exchange for a lower cross-region cost.
- **Pull-on-fault.** The secondary region fetches weights on demand on a cold-start path. Not acceptable for tier-1; the pull time would be inside the RTO. Reserved for tier-3 batch workloads where restart latency is tolerable.

### 3.4 Content integrity on replication

A weight that silently differs between regions produces a split-brain quality signal — half the responses are wrong — and is maximally hostile to debug. The function enforces:

- **Content hash** stamped on the artifact on upload. The hash is in the object metadata.
- **Hash verification** in the serving tier's bootstrap: the serving process refuses to start if the loaded hash does not match the pinned hash.
- **Replication integrity check** after replication: the cross-region replicated object's hash is compared to the source; mismatch triggers an alert and the destination is treated as unpopulated.

The three checks together prevent the subtle corruption class (bit-flip on wire, partial-upload replay) that every vendor explicitly cites as rare but possible. `mod-110` chapter 02's audit discipline retains the hash lineage for the lifetime of the pinned version.

### 3.5 The companion artifacts

Weights are the biggest artifact, not the only one. The function enumerates and replicates:

- **Tokenizer files.** Small (MBs) but required; the serving tier does not boot without them.
- **Model-card / provenance metadata.** Required for the audit record on every served response (`mod-112` chapter 01 §7).
- **Adapter weights (LoRA, soft-prompt).** Same large-object discipline as base weights; often stamped with their own hashes.
- **Prompt registry entries** (`mod-112` chapter 01 §3) — must be resolvable in the secondary region.
- **Runtime config CRDs** (`mod-112` chapter 01 §4) — propagated via GitOps or equivalent; the secondary region must apply the same config version.

A failover that lands weights and misses one of the companions produces a region that "has the model" but cannot serve it. The ISCP's replication checklist enumerates all of them.

## 4. Warm-standby regions for cold-start-sensitive workloads

Cold-start cost dominates RTO on large models. The function solves it with warm standby — a scaled-down functional copy of the serving tier in the secondary region that is always running — and sizes the warm capacity against the failover-traffic estimate.

### 4.1 Why warm standby, not pilot light

AWS's four strategies (chapter 01 §3.1) place pilot light and warm standby on adjacent rungs. The difference is whether the critical path's services are *running* (warm standby) or only *provisioned, not running* (pilot light). On a classical web service the distinction can be minutes of restart latency and is an economic choice. On an AI serving tier the distinction is often the entire RTO budget:

- A 70B-parameter model loading from a cold start on 8× A100 can take 5–15 minutes of shard-file reads, memory allocation, and warmup — numbers vary by hardware, serving stack, and tensor-parallel configuration and must be measured per fleet. <!-- needs-research: validate the model-load-time range against the function's own fleet measurements; a published benchmark is not a substitute for on-fleet timing -->
- A GPU fleet autoscaler often requires minutes of node provisioning before any pod can be scheduled.
- A prefix cache (`mod-104` chapter 06) is empty on a cold region; first-request latencies are elevated for the warm-up window.

A pilot-light secondary that must cold-start on failover has an RTO floor of the sum of these. A warm-standby secondary where the pods are running, the weights are mmap'd, and some traffic is flowing through (the lab chapter 05 §3 writes the cold-path exercise mechanic) has an RTO floor of the cutover latency plus the capacity-scale-out time.

### 4.2 Sizing the warm capacity

The function sizes warm-standby capacity against a defined traffic-shift model. Three sizing bands:

- **Full absorption.** The secondary is sized to serve 100% of primary traffic. Cost: two full-scale fleets. Typical only for the highest-tier workloads where the failover RTO target is sub-minute.
- **Partial absorption with scale-out.** The secondary is sized to serve 30–50% of primary traffic immediately and scales to full over 10–20 minutes. Cost: ~1.3–1.5× single-fleet cost. The function's default for tier-1 workloads.
- **Pilot light with named hot cores.** A subset of the critical path (the smallest model in the ensemble, the safety classifier) runs warm; the primary model itself is pilot-light with pre-staged weights. Lower cost; longer RTO. Acceptable only when the BIA explicitly sized the longer RTO.

The sizing decision is in the ISCP and re-read annually against the BIA (chapter 06 §4).

### 4.3 Keeping warm capacity warm

A warm standby that has not served real traffic for weeks is functionally cold. The function runs continuous validation:

- **Synthetic traffic.** A small, constant synthetic load (chapter 05 §3) exercises the secondary end-to-end — weight resolution, inference path, downstream dependencies, trace emission.
- **Shadow traffic.** A fraction of primary traffic is mirrored to the secondary and discarded (`mod-111` chapter 03). This also serves as the ongoing quality signal — a judge-score divergence between primary and shadow indicates a weight or config drift before it manifests in a failover.
- **Scheduled region-switch drills.** The function switches traffic to the secondary for a short, scheduled window (chapter 05 §3.2). An untested warm standby is an unproven warm standby.

### 4.4 The warm-standby SLI set

The warm-standby fleet has its own SLI stack, distinct from the primary's. The function monitors:

- `warm_standby_ready{region}` — a boolean composite: weights resolvable, pods running, synthetic traffic succeeding.
- `warm_standby_capacity_ratio{region}` — current secondary capacity as a fraction of primary traffic. Alerts when below the sizing-band floor.
- `warm_standby_quality_delta{region}` — the primary / shadow judge-score delta. Alerts when non-zero in steady state.
- `warm_standby_weight_lag_seconds{region}` — time since the latest promoted weight version was last verified resolvable in the secondary.

Each metric has an alert threshold and a runbook link.

## 5. Provider-level fallback across regions

The fourth substrate is the case where the primary failure is the inference provider itself. A hosted provider (`mod-107` chapter 01) is an external dependency; its regional posture is not under the function's control. The DR plan must include a provider-level failover path.

### 5.1 The two provider-regional failure shapes

- **Provider regional endpoint red.** The provider's `us-east-1` endpoint returns 5xx or stalls while the global status page reports healthy. A single-provider customer has no mitigation other than self-shifting to the provider's other regional endpoint.
- **Provider platform regional outage.** The provider has a region-wide incident on its own substrate. The provider's other regions are healthy; the global API may fail over automatically, may not.

### 5.2 The provider-fallback DR extension

`mod-107` chapter 02 writes the general provider-fallback contract — health signal, equivalence table, policy, semantics. The DR extension adds:

- **Regional affinity per request.** Every inference call is tagged with the provider region it hit. Trace / metric emission includes `provider_region` as a first-class label.
- **Region-level circuit breaker.** The `mod-107` chapter 02 §2.1 error-rate breaker is scoped to `(provider, region)`, not just `provider`. A sustained failure in `anthropic/us-east-1` opens the breaker for that pair without disabling the provider globally.
- **Preferred-region list per route.** Each route has an ordered list — primary provider's primary region, primary provider's other region(s), fallback provider's region(s) — and the router follows the list on breaker open.
- **Residency-aware preferences.** The preferred-region list for an EU route may not list a US region; chapter 04 writes the residency constraint.

### 5.3 The provider-regional playbook step

On `rb-regional-failover`, the function:

1. Identifies which substrates in the region are failing — cloud infra, model provider, both.
2. For the model-provider case, confirms the provider's status and whether the provider's own cross-regional failover has already shifted traffic.
3. If the provider has not shifted, invokes the router's region-level break — either by manual toggle in the router's control plane or by letting the error-rate breaker trip.
4. Watches `mod-107` chapter 04's provider-status ingestion for the regional recovery signal; closes the breaker on recovery per chapter 02 §4.

The playbook is specific to the provider's primitives. Anthropic, OpenAI, Bedrock, Vertex, and Foundry each publish different region-level controls and recovery semantics; the function reads the vendor's current documentation and records the specifics in the per-service ISCP. <!-- needs-research: on first ISCP authoring, pull each provider's current regional-failover documentation and cite the specific regional endpoints the service depends on -->

## 6. The end-to-end regional-failover sequence

The playbook composes the four substrates into a single ordered sequence. The on-call engineer reads it top to bottom on a page.

### 6.1 Detect

- The three-family alert (`mod-109` chapter 05 §2) fires on serving, quality, or cost in a specific region. Alert routing (`mod-104` chapter 05) includes the region in the alert payload.
- The `mod-108` chapter 02 incident-command structure declares the incident and names the IC.

### 6.2 Decide

The IC decides between hold, partial-shift, or full-failover based on:

- The impairment class from §1 (full loss, partial, degraded-200, control-plane-only).
- The time elapsed vs. the service's RTO budget.
- The health of the secondary (per the §4.4 SLIs).
- The residency and compliance constraints from chapter 04.

The decision is logged in the incident channel with the inputs.

### 6.3 Execute

- Invoke the endpoint-failover substrate (§2) — DNS record change, Anycast withdrawal, or global-LB backend disable.
- Invoke the provider-level break (§5) if the provider substrate is the failing component.
- Confirm the secondary is absorbing traffic — SLI stack, dashboards, synthetic probes.
- Scale the secondary if the §4.2 sizing requires post-failover scale-out.

### 6.4 Preserve state

- Conversation / memory state (chapter 03): confirm the state store is readable from the secondary region; residency-respecting reads apply.
- In-flight streams: expect loss. The client-side retry semantics should surface the loss to the user; the agent-system idempotency discipline (`mod-105` chapter 05) applies.
- Audit: the audit emission pipeline must survive the failover; `mod-110` chapter 05 retention does not pause for a DR event.

### 6.5 Communicate

Per chapter 06 §5's communication templates and `mod-108` chapter 06's legal / compliance handoff. Status-page update, customer notice, internal stakeholder update, regulator notice (if applicable).

### 6.6 Recover

- Primary recovery signal (vendor status, local health checks, dashboard return).
- Decision on fail-back: immediate (small RPO delta, clean primary), scheduled (burned-in primary, non-peak window), or hold (primary not fully trusted; stay on secondary pending root cause).
- Fail-back follows the reverse of §6.3; the playbook includes an explicit fail-back section, not just a failover section.

### 6.7 Post-mortem

`mod-108` chapter 03's blameless post-mortem discipline applies. The DR event's post-mortem feeds the drill programme (chapter 05): a detection gap becomes a drill scenario, an authority ambiguity becomes a decision-rights clause in the ISCP.

## 7. The regional-failover checklist

Before enabling the playbook on a service, the function confirms:

1. **Impairment class taxonomy** is mapped to named runbooks per §1.5.
2. **Endpoint-failover substrate** is chosen and tested per §2.
3. **Weight replication** meets the per-service weight-propagation RTO per §3.3, with hash integrity checks per §3.4.
4. **Companion artifact replication** (tokenizers, metadata, prompt registry, config CRDs) is enumerated and tested per §3.5.
5. **Warm-standby sizing band** (full absorption, partial with scale-out, pilot-light hot cores) is chosen and documented per §4.2.
6. **Warm-standby SLI stack** is live and alerting per §4.4.
7. **Provider-regional affinity** is tagged and the provider-region breaker is wired per §5.2.
8. **Preferred-region list** per route is residency-checked against chapter 04.
9. **End-to-end sequence** (detect, decide, execute, preserve state, communicate, recover, post-mortem) is drilled per chapter 05.
10. **The ISCP for the service** (chapter 06 §3) names each of the above and cites the vendor documentation it rests on.

A service that cannot pass the checklist has an aspirational DR posture, not a tested one. The next drill (chapter 05 §2–§4) is the opportunity to fix it, not to discover it.

## Summary

- A regional outage comes in four shapes — full loss, partial impairment, degraded-200, control-plane-only — and each has a named runbook; the playbook does not optimise only for the full-loss case.
- Endpoint failover uses DNS-based routing, BGP Anycast, or a global load balancer. The function picks per service against detection latency, cutover semantics, and streaming fit. Anycast and global LB are the typical choices for streaming AI inference.
- Model weights replicate cross-region via S3 CRR / Cloud Storage dual-region / Azure GRS; the weight-propagation RTO is a first-class planning metric. Pre-staged replication is the default on tier-1; lazy or pull-on-fault patterns are reserved for lower tiers.
- Warm standby — not pilot light — is the correct secondary posture for cold-start-sensitive models. Sizing is explicit (full absorption, partial with scale-out, pilot-light hot cores) and the warm fleet runs synthetic + shadow traffic + scheduled drills to stay warm.
- Provider-level fallback extends the `mod-107` chapter 02 contract with regional affinity, region-scoped breakers, and residency-aware preferred-region lists. Each provider's regional primitives are read per vendor.
- The end-to-end sequence — detect, decide, execute, preserve state, communicate, recover, post-mortem — is the ordered sequence on the on-call page. A ten-item checklist is the gate from "playbook drafted" to "playbook enabled."

Chapter 03 writes the state-reconstitution plan — the agent-state, KV, tool-call, and trace replication that the playbook's "preserve state" step depends on.
