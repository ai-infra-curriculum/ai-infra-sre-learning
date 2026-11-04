# Peer Boundaries — Where Capacity Planning Stops

## Why this chapter exists

The AI SRE's capacity-planning role sits at the intersection of three deeper disciplines: GPU performance engineering (how a kernel runs on a specific generation), inference-engine engineering (how the engine schedules and batches), and base Kubernetes autoscaling (how pods and nodes respond to signals). Each is a full role at this level 35 track's peer boundaries — the `ai-infra-performance` peer, the `ai-infra-inference-engineer` peer, and the `ai-infra-engineer` role at level 20. **Capacity planning does not do their work.** It consumes their outputs, operates the fleet under their configuration choices, and routes to them when the capacity symptom is actually a depth problem in their domain.

The reliability pattern at play is not new — `mod-101` chapter 02 and `mod-104` chapter 07 named it: the AI SRE role is a *generalist with routing competence*, not a specialist with ownership of every layer. The point of a module on capacity planning is that it must still fit inside that boundary. Chapter 07 draws the boundary explicitly so the capacity plan and the forecast do not accidentally trespass into kernel tuning or engine engineering.

## 1. The three peer boundaries

### 1.1 The `ai-infra-performance` peer (GPU kernels and systems)

**Owns.** Kernel-level throughput on each hardware generation. Attention kernel choice (FlashAttention 2 vs. FlashAttention 3 vs. cuDNN SDPA), MoE kernel optimisation, FP8 / FP4 quantisation kernels, GEMM auto-tuner selection, kernel-to-SM scheduling, warp-level primitives. Nsight Compute and Nsight Systems profiling. Tensor-memory-accelerator (TMA) and Blackwell's new architectural features. CUDA toolkit / cuDNN pins and the kernel-regression failure class (`mod-105` chapter 03).

**What capacity planning consumes.** The *measured throughput curves* of the engine on each hardware generation. "Prefill tokens/s per H200 at TP=4, FP8 weights, batch 64, context 2K" is a number produced by the performance peer, consumed by the forecast (chapter 02) and the autoscaler target calibration (chapter 05).

**When to route to them.** The symptom looks like capacity — "we need more replicas to hold TTFT" — but the root cause is a kernel regression. Signals: identical hardware, identical engine version, identical workload, but throughput dropped between deploys. The fix is in the kernel; adding replicas is a band-aid.

**What capacity planning does not do.** Pick kernels. Tune Nsight-measured hotspots. Decide between FlashAttention variants. Benchmark at the kernel level. All of those are the performance peer's work. The AI SRE's forecast consumes "here are the throughput numbers at the operating point we tune for" and does not question whether a different kernel would be faster.

### 1.2 The `ai-infra-inference-engineer` peer (serving engines)

**Owns.** vLLM / SGLang / TensorRT-LLM / TGI / Triton engine selection and configuration. Scheduler knobs — `max_num_seqs`, prefill-token budget per step, continuous-batching parameters. Prefix-cache implementation and sizing. Chunked prefill, prefill/decode disaggregation, speculative decoding, MultiLoRA. Engine upgrades and compatibility matrices.

**What capacity planning consumes.** The *engine-exposed metrics* that the autoscaler and the SLO stack read (queue depth, KV-cache utilisation, in-flight sequence count), the *scheduler-side knob values* that define the replica's operating envelope (`max_num_seqs`, prefill budget), and the *engine's throughput curves* at those knob values.

**When to route to them.** The symptom is a scheduler-side pathology that the capacity plan cannot fix — HOL blocking because chunked prefill is misconfigured, prefix-cache hit rate collapsed because of an engine version change, speculative decoding acceptance rate drifted. Signals: the forecast's inputs did not change, the fleet size did not change, but the TTFT distribution shifted or the effective throughput dropped. The fix is in engine configuration or engine code; capacity adjustments will not restore the previous operating point.

**What capacity planning does not do.** Pick between vLLM and SGLang. Tune `max_num_seqs` from first principles. Design the speculative-decoding draft model. Write the chunked-prefill scheduler. All of those are the inference-engineer peer's work. The AI SRE's forecast consumes "the engine runs with these knobs at these throughput curves" and does not re-derive the knobs.

### 1.3 The `ai-infra-engineer` peer at level 20 (base Kubernetes)

**Owns.** Base Kubernetes cluster operation — control plane, networking (CNI, service mesh, ingress), storage (CSI, persistent volumes), scheduler (default scheduling, taints / tolerations, affinities), base autoscaler patterns (HPA on CPU, cluster autoscaler, VPA, PDB). GPU device plugin, NVIDIA GPU operator, Kueue for job scheduling. Karpenter installation and `NodePool` CRD authoring from the base operator perspective.

**What capacity planning consumes.** The cluster primitives that the LLM-shaped autoscaling pattern (chapter 05) builds on — the HPA controller, Karpenter, Prometheus Adapter / KEDA, priority classes, pod disruption budgets. The pattern is layered: the AI SRE's HPA config references custom metrics; those metrics come from the engine (inference-engineer peer); the autoscaler plumbing is base Kubernetes (`ai-infra-engineer` peer).

**When to route to them.** Signals: the HPA is not responding to metric changes (Prometheus Adapter misconfigured, metric name wrong at the API layer); Karpenter is not provisioning nodes (quota in the cloud provider, missing permissions, misconfigured `NodePool`); pods pending because of scheduling constraints that are not AI-specific (taints, affinities, resource requests misstated). The AI SRE should recognise the symptom as a Kubernetes-primitive issue and route; they should not spend an incident hour debugging the prometheus-adapter config when the base-K8s peer owns it.

**What capacity planning does not do.** Install the cluster. Write the CNI config. Author the base `EC2NodeClass` resource. Decide between node-pool managers. Those are the `ai-infra-engineer` peer's work. The AI SRE's autoscaler plan consumes "the cluster has Karpenter running and a working Prometheus Adapter" and does not re-install them.

## 2. How to recognise which peer owns an incident

The capacity signal is nearly always the *leading* signal — an SLO is burning, a queue is deep, the forecast says the fleet should be fine. The question at 3 AM is: where does the fix live? A rough decision tree:

### 2.1 Did the workload shape change?

- **Prompt length distribution shifted.** The forecast's input changed. Capacity planning problem — size up, re-forecast, communicate with the product team about the underlying driver.
- **Completion length distribution shifted.** Same — capacity planning.
- **Prefix-cache hit rate dropped.** First check if the engine's cache is healthy (route to inference-engineer); if yes, the template changed (route to application layer, `mod-105` chapter 05).

### 2.2 Did the fleet shape change?

- **A hardware generation was swapped.** If throughput curves on the new generation do not match the forecast, call the performance peer to confirm the measured throughput; also re-run the Pareto curve (chapter 06).
- **An engine version was promoted.** Call the inference-engineer peer; check the engine's changelog for scheduler changes, prefix-cache changes, default knob changes.
- **The cluster changed (CNI upgrade, scheduler change, cordon action).** Call the base-K8s peer.

### 2.3 Did a signal start reading differently?

- **Queue-depth gauge jumped without a traffic change.** Probably an engine-side scheduler issue — call inference-engineer.
- **GPU-util jumped without a throughput change.** Possibly a kernel change — call performance.
- **Prometheus Adapter stopped exposing the metric.** Base-K8s issue — call the base-K8s peer.

### 2.4 Is the signal just capacity?

- **Traffic rose past forecast, fleet sized correctly, autoscaler responding, SLO holding.** No problem — the system is working.
- **Traffic rose past forecast, fleet sized correctly, autoscaler responding, SLO burning.** Re-run the Pareto curve; autoscaler target may need tightening. Capacity planning.
- **Traffic as expected, fleet undersized, autoscaler responding but limited.** Forecast miss — update the forecast, re-size, communicate. Capacity planning.
- **Traffic as expected, fleet sized correctly, autoscaler NOT responding, SLO burning.** The autoscaler plumbing is broken somewhere — chase the signal path, route to the right peer at the broken link.

## 3. The shared artifact chain

The three peer boundaries are not isolated silos; they produce artifacts the AI SRE's capacity plan consumes and depends on.

### 3.1 From the performance peer

- **Per-model, per-hardware throughput curves** at each generation your fleet uses. Produced from kernel-level benchmarks and engine integration tests.
- **Expected performance deltas** for planned hardware swaps, framework upgrades, or quantisation changes. Published in advance of the rollout.
- **Regression triage** for performance incidents where capacity is the symptom.

### 3.2 From the inference-engineer peer

- **Engine configuration manifests** per workload — the knobs that define the replica's operating envelope.
- **Prefix-cache behaviour documentation** — how the cache is sized, when it is invalidated, what the hit-rate reporting means.
- **Engine-upgrade readiness assessments** — "upgrading to vLLM X.Y changes the default `max_num_seqs` and tightens the KV-cache allocation; recalibrate the forecast."
- **Scheduler-side incident triage** when capacity symptoms point at engine behaviour.

### 3.3 From the base-K8s peer

- **Cluster SLOs** — the base platform's own reliability contract (control plane availability, scheduler latency, Karpenter provisioning time).
- **Autoscaler plumbing readiness** — prometheus-adapter / KEDA configured, custom metrics API exposed, Karpenter `NodePool` CRD set, GPU device plugin healthy.
- **Cluster-change advance notice** — upgrade windows, scheduler changes, CNI replacements. These can change the capacity math.

The capacity plan's artifact (chapter 02 §5), the autoscaler plan (chapter 05 §8), and the verification artifact (chapter 06 §9) cite these three upstream artifacts. A capacity plan whose throughput curves are not sourced from the performance peer, or whose engine knobs are not sourced from the inference-engineer peer, is ungrounded and will break on the first re-measurement.

## 4. Where capacity planning is distinctly the AI SRE's role

The boundaries are drawn to leave a clear role for the AI SRE. Capacity planning *is*:

- **Forecasting.** Projecting demand against a growth model, shape distributions, and prefix-cache assumptions.
- **Aggregating.** Composing per-workload-class demands into a fleet-level requirement with explicit headroom.
- **Translating to node-pool shape.** Deciding GPU generation mix, instance types, pool roles (baseline / burst / batch), and reservation calendar.
- **Quota planning.** Provider-side capacity as a first-class artifact.
- **Operating the autoscaler.** Choosing the signals, setting the targets (from measurements the performance and engine peers produce), running the plan under production.
- **Verifying.** Running load tests as the closed-loop verification; producing the go/no-go gate.
- **Communicating.** To product, to finance, to leadership. The AI SRE owns the "what does the fleet cost, what does it do, where will it break" story at the business cadence.

Everything above requires the peer artifacts but is not inside any peer's role. The AI SRE is the *integrator* and the *operator* of the capacity envelope; the peers are the *providers* of its inputs.

## 5. A worked example of the peer handoff

Illustrative scenario: the capacity-verification load test reveals TTFT p99 has regressed 25% versus the previous quarter's test, at the same workload shape. The forecast said the fleet should hold. What now?

**Step 1 — rule out workload shape.** Compare the current test's prompt / completion distributions to last quarter's. If they differ, the test input changed and the regression is apparent, not real — fix the test. Assume they match.

**Step 2 — rule out fleet shape.** Replica count, hardware generation, engine version, model version. If any differ, isolate to the changed variable.

- Engine version changed? Route to inference-engineer: "did this upgrade change the scheduler's default behaviour?"
- Hardware generation shifted? Route to performance: "did the H200 rollout include a kernel version that regressed decode throughput on this model?"
- Model version promoted? Route to ML platform / inference-engineer: "did the new quantisation change the per-token KV-cache footprint, reducing effective `max_num_seqs`?"

**Step 3 — rule out plumbing.** Prometheus Adapter healthy? HPA responding? Node-pool provisioning? If any broken, route to base-K8s peer.

**Step 4 — accept it is a forecast miss.** If all three above are clean, the capacity forecast was optimistic — more concurrent long-context requests than projected, prefix-cache hit rate lower than expected. Update the forecast, size up, re-run the Pareto curve.

The pattern: the AI SRE eliminates the capacity-shaped explanations, routes the ones that belong to peers, and owns the forecast miss if that is what remains. All four steps complete in the first incident hour; none involves tuning a kernel, writing an engine patch, or reinstalling a cluster addon.

## Summary

- **Capacity planning is a generalist integrator role.** It consumes performance, engine, and base-K8s peer artifacts; it does not re-derive them.
- **Three peer boundaries matter.** `ai-infra-performance` (kernels and hardware), `ai-infra-inference-engineer` (engines and scheduling), `ai-infra-engineer` level 20 (base Kubernetes and autoscaler plumbing).
- **The decision tree at 3 AM is routable.** Did the workload change (capacity), the fleet change (route to the right peer), the signal change (chase the signal path)? The AI SRE eliminates in that order and owns what remains.
- **The shared artifact chain is the integration surface.** Throughput curves from performance, engine knobs from inference-engineer, cluster plumbing from base-K8s, feed the forecast, autoscaler plan, and verification artifact.
- **What the AI SRE owns distinctly** is the forecast, aggregation, translation to node-pool shape, quota planning, autoscaler operation, verification, and the business-cadence communication. That is the role; the peers backstop the depth.

This closes the capacity-planning module. The next module in the track (`mod-107`) addresses the other half of the provider-side capacity conversation: vendor evaluation, SLA negotiation, and model-provider risk.
