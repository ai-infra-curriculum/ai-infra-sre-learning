# Workload-Aware Autoscaling for LLM Serving

## Why this chapter exists

Classical Kubernetes autoscaling patterns — HPA on CPU utilisation, Cluster Autoscaler on unschedulable pods, Karpenter on spec-driven node provisioning — are the AI SRE's baseline knowledge from the `ai-infra-engineer` peer at level 20. They are **necessary but inadequate** for an LLM serving fleet, because the metrics they default to (CPU %, memory %, request rate, even GPU utilisation) are weak predictors of the user-visible signal the SLO cares about (TTFT, TPOT, error-budget burn). Scaling an LLM fleet on CPU% scales on the wrong axis; scaling on GPU% often scales too late because by the time GPU% says "overloaded" the engine's queue is already deep enough to burn TTFT.

This chapter builds the autoscaling discipline that is specific to LLM workloads: which signals to scale on, what thresholds to pick and why, what cold-start behaviour to accept and how to mitigate it, when warm pools are worth the cost, and how to share spot capacity across workloads without creating a correlated-failure shape. The chapter stays on the capacity-planning side of the boundary — the engine's internal autoscaling knobs (dynamic batch size, prefill/decode balance, cache-eviction policy) belong to the inference-engineer peer; this chapter targets the signals Kubernetes and the cluster autoscaler can read.

## 1. The signal problem

An autoscaler's job is to add capacity before SLO burn and remove it after the surge subsides, with enough stability that the fleet is not oscillating. The signal the autoscaler reads must satisfy three constraints:

1. **Leading.** The signal must rise before the SLO-bound metric regresses. A lagging signal triggers scale-up during the incident, not before.
2. **Specific.** The signal must be sensitive to the actual bottleneck. A signal that rises for unrelated reasons triggers scale-ups you do not need (and scale-downs you do).
3. **Stable.** The signal must not oscillate. A signal that moves with every request burst makes the autoscaler flap.

The classical signals score badly against all three:

### 1.1 Why CPU % fails

LLM inference is GPU-bound. CPU is involved in the request-handling path (tokenisation, scheduler, response serialisation, log emission) but CPU % routinely sits at 20–40% on a replica that is fully GPU-saturated. CPU % is:

- **Lagging.** CPU rises only if tokenisation / response handling is the bottleneck, which is rare.
- **Wrong direction.** CPU can rise when decoding-batch size drops and more request-handling overhead per decode step is incurred. It is confounding.
- **Unstable.** CPU % jitters with background noise.

Scaling on CPU is strictly worse than not autoscaling for an LLM fleet. Do not do it.

### 1.2 Why GPU % fails (as the primary signal)

GPU % — specifically, `DCGM_FI_DEV_GPU_UTIL` or the `nvidia-smi` utilisation.gpu field — measures the fraction of time at least one kernel is executing on the SMs. It tells you *the GPU is doing something*, not *the GPU is close to its bottleneck*. For an LLM fleet:

- On a decode-bound replica serving a reasonable batch, GPU % reads 95%+ *at any throughput*, because decode steps run back-to-back regardless of batch fullness. GPU % says the replica is "pegged" when the replica has 80% of its decode-batch slots free.
- On a prefill-bound replica, GPU % may read 70–85% at saturation because prefill has gaps (compute-memory alternation). GPU % says the replica has headroom when it is actually full.
- GPU % is insensitive to context length, batch composition, and prefix-cache hit rate — all of which change the replica's effective capacity.

GPU % is **useful as an idleness signal for scale-down**: a sustained GPU % below some threshold (say, 20%) over a long window is reasonable evidence that a replica is underused. It is **not useful as a scale-up signal** for the reasons above.

### 1.3 The right signals

Three signals that satisfy leading / specific / stable for LLM fleets:

- **Queue depth.** The engine's gauge of requests waiting for admission into the batch. Leading (rises before TTFT regresses), specific (rises exactly when the fleet is bound), stable (smoothed over a short window). Exposed by vLLM as `vllm:num_requests_waiting`, SGLang as its queue gauge, TGI as `tgi_queue_size`, TensorRT-LLM / Triton as its scheduler queue. This is the default autoscaling signal for most production fleets.
- **TTFT p99 (or TTFT p95).** The SLO-bound latency, high-percentile. A targeted p99 close to the SLO threshold is a direct capacity contract. More lagging than queue depth but more directly tied to the user contract. Works well as a *secondary* signal that catches scenarios queue-depth misses (e.g., long-prompt HOL blocking that stalls TTFT without growing the queue).
- **KV-cache utilisation.** The engine's gauge of KV-cache budget consumed. Rises as in-flight sequences accumulate. Specific to decode-bound saturation; a strong predictor of admission starvation when it crosses 85%.

The autoscaler reads **one or more of these** through Kubernetes custom metrics (Prometheus Adapter, KEDA, custom metrics APIs). Chapter's section 2 gets concrete.

## 2. HPA on custom metrics — the concrete pattern

### 2.1 The custom-metrics pipeline

The standard pattern:

```
engine → /metrics endpoint → Prometheus scrape → prometheus-adapter →
  Kubernetes custom.metrics.k8s.io API → HPA → replica count
```

- **Engine side.** Modern engines expose queue depth natively. For vLLM, the gauge is `vllm:num_requests_waiting`. For SGLang, TGI, and others, verify the current metric name against the engine's docs.
- **Prometheus side.** Scrape the engine's `/metrics`. Keep the scrape interval short (10s or less) so the HPA signal is not lagging on scrape cadence.
- **Prometheus Adapter.** Exposes selected Prometheus queries through the Kubernetes custom-metrics API. The query can transform the raw gauge: smooth over a window, average across pods, slice by workload label.
- **HPA.** References the custom metric by name and target value. Scales the deployment / replica set based on current vs. target.

Alternative path: **KEDA** (Kubernetes Event-Driven Autoscaler) with its Prometheus scaler. KEDA can scale deployments to zero, supports more scaler types, and generally has more flexible behaviours. For LLM workloads where cold start is expensive, scale-to-zero is usually not what you want (see section 4), but KEDA is still useful for its scalers against external queues (SQS, Pub/Sub, Kafka) which matter for the batch-workload path.

### 2.2 The HPA manifest

Illustrative HPA scaling on average queue depth per pod:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: vllm-chat-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vllm-chat
  minReplicas: 6
  maxReplicas: 40
  metrics:
    - type: Pods
      pods:
        metric:
          name: vllm_num_requests_waiting_avg_1m
        target:
          type: AverageValue
          averageValue: "4"
    - type: Pods
      pods:
        metric:
          name: vllm_ttft_p95_seconds
        target:
          type: AverageValue
          averageValue: "1.5"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
        - type: Pods
          value: 4
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
        - type: Percent
          value: 10
          periodSeconds: 180
```

The shape is deliberate:

- **Two metrics.** Queue depth is the primary leading signal; TTFT p95 is the backstop that catches HOL-style scenarios. The HPA scales on `max()` of the two.
- **Asymmetric stabilisation.** Scale-up is fast (30s) because an incident is forming; scale-down is slow (10 min) because unnecessary flaps are expensive.
- **Scale-up policy is aggressive but bounded.** Doubling replicas once per minute is a lot; it is appropriate for sudden surges and must be upper-bounded by `maxReplicas` to prevent runaway costs during a pathological scenario.
- **Scale-down is conservative.** 10% of replicas every 3 minutes keeps the fleet shrinking slowly after the surge passes.

### 2.3 Picking the target values

The target values come from the capacity forecast and the SLO:

- **Queue-depth target.** From Little's Law (chapter 01 §2), the steady-state queue should be near zero when the fleet is correctly sized. A target of "4 requests waiting per pod averaged over 1 minute" is a cushion that says "we are under-provisioned by about one batch-slot per pod." Lower targets (e.g., "1") make the HPA scale up on any transient congestion; higher targets (e.g., "16") let the queue grow before scaling. Pick the target so that `target_queue_depth × TTFT_at_target_depth < TTFT_SLO - headroom`.
- **TTFT target.** Below the SLO by a fixed headroom. If the SLO is TTFT p95 ≤ 2s, the HPA target might be 1.5s — scaling up when p95 crosses 1.5s gives the autoscaler time to add replicas before the SLO is violated.
- **KV-cache target (if used).** Below the engine's hard limit by a fixed headroom. 75% is a common choice; above that the engine starts refusing admission.

The target is not a theoretical derivation. It is a measurement from the load-testing harness (chapter 06): "at a queue depth of 4 per pod averaged over 1 minute, we see TTFT p95 of 1.4s." That measurement sets the target.

## 3. Karpenter for GPU bin-packing with spot resilience

HPA scales pods; the cluster autoscaler scales nodes. For GPU fleets, Karpenter has displaced the classical cluster autoscaler for many deployments because its declarative provisioning model handles GPU heterogeneity, spot capacity, and bin-packing better.

### 3.1 The Karpenter model

Karpenter reads pending pods from the scheduler, consults a set of `NodePool` and `EC2NodeClass` (on AWS; equivalent CRDs on other clouds) specifications, and provisions a concrete node from the matching instance-type set. For GPU workloads:

- **Node pools scoped to GPU generation.** A `NodePool` for H100 (`g5.*`, `p4d.*`, `p5.*`, etc. depending on the specific SKU), another for H200, another for AMD, another for Trainium. Pods request the generation via node-selectors or affinity, Karpenter provisions from the matching pool.
- **Instance-type lists, not fixed types.** A `NodePool` for H200 lists all SKUs that satisfy the workload requirement (e.g., `p5e.48xlarge`, `p5en.48xlarge` on AWS when they are the current H200 SKUs). Karpenter picks the most-available / cheapest instance from the list, which smooths capacity variability.
- **Spot + on-demand mix.** A `NodePool` can specify capacity type priorities: prefer spot, fall through to on-demand if spot is unavailable. For LLM workloads, the pattern below is more useful than naive spot-first.

### 3.2 GPU bin-packing considerations

Bin-packing matters because GPUs are expensive. Two useful patterns:

- **One replica per GPU, not one replica per node.** A p5.48xlarge has 8 H100 GPUs. Running one replica per node wastes 7 GPUs. Run one replica per GPU (eight pods per node, one GPU each) and let the scheduler bin-pack. The engine must support single-GPU replicas for the model; if tensor parallelism is required (model too big for one GPU), the replica is TP-N and consumes N GPUs. Match the TP degree to the GPU memory budget.
- **Topology-aware scheduling.** For TP replicas, the GPUs must be on the same node (or on the same NVLink domain for NVL72-style setups). Use node-affinity, topology-spread constraints, and the Kubernetes device plugin's NVLink-aware annotations (where supported) to keep TP groups co-located.

### 3.3 Spot capacity for LLM burst

Spot GPU capacity can be substantially cheaper than on-demand, but it can be reclaimed by the cloud provider on 2-minute notice. For LLM fleets, spot is viable for:

- **Burst capacity above baseline.** The baseline fleet is on reserved / on-demand; the burst pool is spot. If spot is reclaimed, traffic shifts back to the baseline which is sized to hold at reduced throughput.
- **Batch workloads.** Batch jobs with checkpointing tolerate spot reclamation cleanly.
- **Rolling capacity during model updates.** New model versions roll out on spot replicas first, promoted to on-demand only after validation.

Spot is **not** viable for:

- **Baseline capacity.** You cannot plan a baseline on a resource that can disappear. The baseline must be on-demand, reserved, or capacity-blocked.
- **Workloads with session affinity to a replica.** Spot reclamation kills the session.

The Karpenter `NodePool` pattern for a spot burst pool:

```yaml
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["p5.48xlarge", "p5e.48xlarge"]
      taints:
        - key: workload-tier
          value: burst
          effect: NoSchedule
```

Pods that tolerate the taint can land on the spot burst pool; pods without the toleration land only on the baseline pool. The baseline and burst pools are different `NodePool` resources; the HPA scales pods, Karpenter responds by provisioning from whichever pool matches the pod's affinity.

### 3.4 Pre-emption-handling

Karpenter supports node interruption handling via the cloud's termination-notice mechanism (e.g., AWS EC2 Spot Instance interruption notice). When a spot node is marked for interruption, Karpenter drains it; pods are evicted and rescheduled. The engine must handle graceful shutdown — stop accepting new requests, drain in-flight, close the server port — within the notice window.

For vLLM / SGLang / TGI, graceful shutdown is a `preStop` hook that calls the engine's shutdown endpoint and waits. Verify the engine's shutdown behaviour under load against the interruption window before relying on spot in production.

## 4. Warm pools and cold-start sensitivity

LLM cold start is expensive. Loading a 70B-parameter model into GPU memory from a container image takes minutes: pull the container (tens of GB), download the model weights if not baked in, load into GPU memory, warm the engine's compiled kernels, run a warmup batch. Cold start times of 3–10 minutes per replica are typical and not easily shortened without substantial engineering (model-weight baking into AMIs, persistent volumes, remote-tensor loading, snapshot-based startup).

For capacity planning, cold start creates a gap: the HPA decides to scale up, Karpenter provisions a node, the pod schedules, and only *then* does the cold start begin. The fleet is under-capacity for the duration.

### 4.1 Warm-pool patterns

Three common mitigations:

- **Keep minReplicas above zero.** The HPA's `minReplicas` floor keeps a baseline of replicas always running. Simple and reliable. The floor must be sized to hold the floor-of-traffic (lowest hour of the week). Scaling up is then additive from a non-zero base; the cold-start pain is only on the upscale, not on the every-request-after-idle.
- **Pre-warmed stand-by replicas.** A pool of replicas kept in a warm (but not serving) state, ready to be rotated into service in seconds rather than minutes. Implementation varies; the engine's cold-start behaviour determines what "warm" means (is the model loaded in GPU memory? is the first forward pass pre-run?). Operationally complex; use only when the cold-start cost is unacceptable.
- **Scale-to-near-zero, not to zero.** For very bursty workloads with long idle periods, scale down to a floor of 1–2 replicas that serve the trickle and absorb a surge in O(seconds) rather than O(minutes). The floor cost is the cost of insurance.

### 4.2 When scale-to-zero is correct

Scale-to-zero is appropriate when:

- The workload is genuinely intermittent with cold-start-tolerant latency SLOs (e.g., "first request after idle can take 60s").
- The cost of keeping replicas warm exceeds the cost of cold starts over the long run.
- The platform supports it well (KEDA with Knative, specifically for request-driven scale-to-zero).

For most production LLM serving fleets, scale-to-zero is **not** the right pattern because the SLO on TTFT is seconds, not minutes. Scale-to-near-zero (`minReplicas: 1` or `2`) is the common compromise.

## 5. Knative as a compare-point

Knative Serving offers a serverless-shaped autoscaler (`KPA`, the Knative Pod Autoscaler) that scales on concurrency or RPS and supports scale-to-zero natively. Knative's model is cleaner than HPA-plus-custom-metrics for some workloads; it is worse for others. The AI SRE should know where it fits.

### 5.1 Where Knative is a good fit

- **Small, fast-loading models** where cold start is seconds, not minutes. Knative's scale-to-zero becomes free.
- **Request-driven workloads with clean concurrency semantics**, where requests are short-lived and independent. Knative's concurrency-based scaling (`containerConcurrency`) is a natural fit.
- **Multi-model routing on shared infrastructure**, where Knative's traffic-splitting and revision model helps manage model versioning.

### 5.2 Where Knative is not a good fit

- **Large models with long cold starts.** Scale-to-zero is a loaded footgun; every cold request takes minutes.
- **Continuous-batching engines where concurrency is managed inside the pod.** Knative's `containerConcurrency` and the engine's `max_num_seqs` fight each other for control of batch size; Knative queues requests at the ingress before they reach the engine's scheduler, which is the wrong layer.
- **Workloads that need custom metrics (queue depth, KV-cache utilisation)** that Knative's default scaler does not read natively.

The default for large-model inference remains HPA + custom metrics + Karpenter. Knative is a legitimate compare-point and the right choice for small-model surfaces (embedding models, small classifiers, lightweight tool-use models) that live alongside the large-model fleet.

## 6. The batch workload as a shock absorber

A production fleet usually has interactive and batch workloads. The batch workload is the autoscaler's secret weapon: batch jobs are latency-insensitive and can be *preempted* by interactive demand. This is the classical "mixed-criticality pool" pattern applied to LLM serving.

The pattern:

- Baseline interactive pool with on-demand / reserved nodes.
- Shared burst pool runs batch when interactive demand is low; batch jobs are drained when interactive HPA scales up.
- Priority classes and `PodDisruptionBudgets` enforce that batch yields to interactive.
- The forecast's burst requirement is partially offset by the fact that batch *absorbs* idle capacity and *releases* it on demand.

Implementation: use Kubernetes `PriorityClass` to rank interactive > batch; use pod anti-affinity or taints to isolate the pools; rely on Karpenter's consolidation to pack batch into the gaps. The savings over separate pools can be substantial — but only if the engine supports graceful preemption of batch requests (either in-engine draining or at the queue level).

## 7. The failure modes to watch

Autoscalers fail. The common modes for LLM fleets:

- **Flapping on noisy metrics.** The signal has too little smoothing; the HPA scales up, the surge subsides, the HPA scales down, the next micro-surge triggers it again. Fix: longer window on the Prometheus query, longer stabilisation window on the HPA, both.
- **Scale-up during cold-start-storm.** A traffic spike triggers scale-up; the new replicas are cold; the cold-start overhead of 10 pods coming up at once overwhelms the pull of weights from the registry; replicas take twice as long to serve. Fix: cap scale-up velocity; pre-stage weights in a shared PVC or an on-node cache; limit image-pull concurrency.
- **Scale-down during a quiet surge.** Load drops briefly, the HPA scales down too aggressively, the next surge finds the fleet under-capacity. Fix: longer scale-down stabilisation window, trailing-window on the scaling metric.
- **Signal collapse during provider-degraded mode.** The provider is slow; your fleet's traffic drops because requests are timing out upstream; queue depth collapses; HPA scales down; when the provider recovers, the fleet is undersized for the recovery surge. Fix: ensure the scaling metric is meaningful when upstream is degraded, or combine queue-depth with an absolute-traffic signal that detects "traffic should be N but we are getting N/2."
- **Karpenter out-of-stock.** The preferred spot SKU is unavailable; Karpenter cannot provision; pods stay pending; the fleet fails to scale. Fix: diverse instance-type lists in the `NodePool`, tested fall-through to on-demand, SNS / Slack alert on `NodeClaim` failures.

Each failure mode is a post-incident input to the autoscaler's tuning. Keep a change log of autoscaler parameters alongside the SLO contract.

## 8. The autoscaling plan artifact

A per-workload autoscaling plan contains:

- **Scaling metric(s).** Primary, secondary, and the Prometheus query that produces each.
- **Target values.** With the measurement from load testing that justifies each target.
- **Min / max replicas.** With the forecast input that produced each bound.
- **Scale-up and scale-down behaviours.** Stabilisation windows and policies, with the reasoning.
- **Node pool(s).** Baseline, burst, batch; capacity types; instance-type lists.
- **Cold-start mitigation.** Warm-pool shape, weight-staging strategy, scale-to-near-zero floor.
- **Cross-pool interaction.** Batch yield behaviour, priority classes, disruption budgets.
- **Known failure modes.** From the list above and from the fleet's own history.

The plan is reviewed with the forecast (chapter 02) and the SLO contract. The three must agree: the autoscaler's targets implement the SLO, the capacity forecast sizes the min/max bounds, and the SLO's burn-rate alerts catch the cases where the autoscaler missed.

## Summary

- **Classical autoscaling signals (CPU, GPU%) do not work for LLM serving.** Use queue depth as the primary leading signal, TTFT p95 / p99 as the secondary backstop, KV-cache utilisation for decode-bound fleets.
- **HPA with Prometheus-adapter or KEDA, on engine-exported metrics.** vLLM's `num_requests_waiting`, SGLang / TGI / TRT-LLM equivalents. Scale up fast, scale down slow.
- **Karpenter is the default node autoscaler for GPU fleets.** Scope node pools by generation, list multiple instance types per pool, use spot for burst only, keep baseline on-demand / reserved.
- **Cold start is expensive.** Rarely scale to zero; usually scale to near-zero. Warm pools and weight-staging are the levers when cold-start cost is unacceptable.
- **Knative is a compare-point.** Fits small-model surfaces; wrong for large-model continuous-batching engines.
- **Batch is the shock absorber.** Preemptible batch workloads absorb idle interactive capacity and release it on demand. Priority classes enforce the yield.
- **Autoscalers fail in characteristic ways.** Flapping, cold-start storms, scale-down after a dip, signal collapse during upstream degradation, Karpenter out-of-stock. Each is an incident class; the autoscaler plan documents mitigations.
- **The autoscaling plan is the operational implementation of the capacity forecast.** The three artifacts — forecast, SLO, autoscaler — must agree; disagreements are the reliability work.

The next chapter closes the capacity loop: the load tests that *verify* the fleet behaves as the forecast and the autoscaler expect.
