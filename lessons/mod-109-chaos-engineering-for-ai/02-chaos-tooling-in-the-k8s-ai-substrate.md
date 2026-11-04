# Chaos Tooling in the Kubernetes AI Substrate

## Why this chapter exists

Chapter 01 wrote the principles. This chapter picks the tools and wires them into the Kubernetes substrate the AI fleet runs on. The point is not to compare feature matrices — that information rots on contact — but to understand which fault-injection *classes* each major tool expresses natively, where the AI-specific gaps are, and how a program composes them so that every experiment in chapter 03's catalogue has a mechanism.

The four tools covered — Chaos Mesh, LitmusChaos, Gremlin, and AWS Fault Injection Service (FIS) — are the ones most commonly deployed on Kubernetes-hosted AI fleets at the scale and risk profile this role operates at. Each is described in its own documentation; this chapter is not a replacement for those docs, it is a routing guide and an integration contract.

The scope is deliberate. A chaos program ultimately executes CRDs (or FIS templates) on a real cluster; the mechanics of installing them and running one experiment live in exercise 02, not in a chapter. What belongs here is the vocabulary (fault classes), the choice architecture (which tool for which class), and the AI-specific integration points (GPU chaos, provider chaos, cache chaos) that neither the tools' docs nor the generic chaos texts cover.

## 1. The five fault classes the K8s substrate exposes

Every major chaos tool for Kubernetes expresses its faults in terms of a shared taxonomy. The names differ — "PodChaos" vs. "pod-kill," "NetworkChaos" vs. "network-loss" — but the physical phenomena being injected overlap almost entirely. An AI SRE should think in these five classes and then map each to the tool's vocabulary at implementation time.

### 1.1 Pod / container chaos

Injects faults at the container runtime: kill, pause (SIGSTOP), container-level OOM, image-pull failure, startup-probe failure, readiness-flap. The *production fault model* this reproduces: eviction under preemption, node-level crash-loop, node drain during rollout, a stuck image pull after a registry regression.

Chaos Mesh: `PodChaos` with `action: pod-kill`, `pod-failure`, `container-kill`. LitmusChaos: `pod-delete`, `container-kill`, `pod-io-stress`, `pod-cpu-hog`, `pod-memory-hog`. Gremlin: Shutdown and Process attacks. AWS FIS: `aws:eks:pod-delete` via the EKS add-on.

AI relevance: `rb-preemption-spike` (`mod-108` chapter 01 §3.4), `rb-ttft-regression` under replica loss, `rb-streaming-stall` under mid-stream pod kill, degraded-fallback drill (chapter 04 of this module).

### 1.2 Network chaos

Injects latency, loss, corruption, duplication, bandwidth cap, and partition at the network layer. Uses Linux traffic control (`tc`) via `netem` under the hood; the tools wrap it in CRDs so it can be targeted at labels / namespaces / specific peers. Also covers DNS: resolver latency, NXDOMAIN injection, hijack to a specific IP. Also covers TLS / HTTP shaping in some tools.

Production fault model: a provider's TCP connection stalls mid-stream (common with upstream overload events), a DNS resolver slows down (a surprisingly frequent AI-ops failure mode because sidecar proxies cache aggressively), the egress NAT drops packets intermittently.

Chaos Mesh: `NetworkChaos`, `DNSChaos`, `HTTPChaos`. LitmusChaos: `pod-network-latency`, `pod-network-loss`, `pod-network-partition`, `pod-dns-error`, `pod-http-latency`, `pod-http-reset-peer`. Gremlin: Latency, Packet Loss, Blackhole, DNS. AWS FIS: `aws:network:route-table-disrupt-cross-region-connectivity`, `aws:network:transit-gateway-disrupt-cross-region-connectivity`.

AI relevance: provider-timeout injection for fallback-contract chaos (chapter 04), embedding-service timeout, judge-model unavailability, retrieval-service network partition, moderation-API delay.

### 1.3 I/O and filesystem chaos

Injects read / write latency, failed syscalls, disk-full conditions, inode exhaustion, and mount-level failures. Reproduces storage degradation and the write-path jitter that shows up on model-weight loads from object storage, index-shard loads from local SSD, and log-write back-pressure.

Chaos Mesh: `IOChaos`. LitmusChaos: `disk-fill`, `node-io-stress`. Gremlin: IO. AWS FIS: EBS volume pause / detach via `aws:ebs:pause-volume-io`.

AI relevance: slow model-weight load (reproduces `rb-weight-corruption`-adjacent timing failures), slow prompt-cache persist, slow log-write under traffic burst, slow retrieval-index shard warm-up.

### 1.4 Time chaos

Shifts the wall-clock or monotonic clock inside a container. Reproduces clock-skew bugs that break TLS validation, token expiry, breaker windows, retry back-off, cache TTL, and — on AI workloads specifically — breaker state machines that time out on a hand-coded `now()` vs. a monotonic clock.

Chaos Mesh: `TimeChaos`. LitmusChaos: `time-chaos`. Gremlin: Time Travel. AWS FIS: not a native primitive.

AI relevance: the fallback-breaker hysteresis window from `mod-107` chapter 02 (if the window reads wall-clock it is a chaos target), vendor-token-expiry flows, prompt-cache TTL eviction windows.

### 1.5 Stress and resource chaos

CPU hog, memory hog, GPU-adjacent memory pressure (filesystem cache contention), bandwidth-saturation. Reproduces the noisy-neighbour, the leaked-allocation, the batch-job contention.

Chaos Mesh: `StressChaos`. LitmusChaos: `pod-cpu-hog`, `pod-memory-hog`, `node-cpu-hog`, `node-memory-hog`. Gremlin: CPU, Memory, Disk. AWS FIS: `aws:ssm:send-command` with stress-ng templates.

AI relevance: KV-cache pressure under memory contention, host-level CPU starvation on the tokenizer path, bandwidth contention affecting NCCL on training.

## 2. The AI-specific gap: no tool has a first-class "GPU chaos" primitive

None of the four tools has a CRD or template called `GPUChaos`. The reasons are legitimate — GPU fault injection runs below the container boundary at the driver / runtime layer, and the tools stay above the container boundary by design — but the consequence is that the AI SRE must **compose** GPU chaos from primitives rather than invoke it as a single call.

### 2.1 GPU-pod chaos by composition

Four building blocks cover most of the AI-specific GPU catalogue:

- **Pod chaos on GPU-pinned replicas.** Kill the serving pod on a GPU host mid-stream; the scheduler re-places it. Reproduces eviction under preemption. The CRD is plain `PodChaos`; the AI-specific part is the selector — label-match on `workload=serving` and `gpu.pool=a100-serving`.
- **Container-level NVML / DCGM failure emulation.** Use an init-container or sidecar in the test replica that intercepts NVML queries and returns a synthetic XID code or ECC counter bump. Chaos Mesh's `HTTPChaos` can intercept DCGM Exporter responses. This emulates `rb-gpu-driver-ecc-xid` without touching the driver.
- **Memory-pressure stress targeted at the inference process.** `StressChaos` with memory-hog pinned to the serving container reproduces KV-cache pressure and triggers the graceful-preemption code path (chapter 03 §4 of this module).
- **Node cordon / drain via a Kubernetes job.** A controlled `kubectl cordon` + `kubectl drain --disable-eviction=false` on a target GPU node reproduces scheduler-initiated eviction. Scripted as a chaos experiment with the same guardrails.

Gremlin's Shutdown attack on a GPU node reproduces a hard-reset case; use sparingly and under tight blast radius.

AWS FIS has an EKS-integrated path that can use the Nitro System's instance-stop primitive to simulate a GPU-instance failure at the hardware boundary. This is the closest any of the four tools come to native GPU chaos, and it is AWS-only.

### 2.2 Driver-level and ECC chaos

Driver-level faults (real XID injection, real ECC errors) are not injectable from Kubernetes. The AI SRE either:

- **Accepts emulation via the observability surface** (counter bump, DCGM response synthesis) and validates the mitigation path from the alert forward.
- **Runs a vendor-lab chaos in staging** when the vendor TAM provides a hardware-fault fixture. NVIDIA's DCGM diagnostics can set some nodes into a "faulty" simulated state; the specific tooling depends on the hardware vendor and is out of scope for a Kubernetes-chaos tool.

A mature program documents which GPU faults are tier-1 injectable (via composition), which are tier-2 (emulation only), and which are tier-3 (vendor-lab only). The policy lives in the chaos-program document (chapter 05 §5).

## 3. The four tools — what each is really for

Pick tooling against the chaos-program's actual catalogue, not against feature-matrix completeness. These are the shapes to internalise:

### 3.1 Chaos Mesh

- **Shape.** A CNCF project, Kubernetes-native CRDs, a web dashboard, a workflow engine for chained experiments, a schedule CRD for continuous experiments.
- **Strengths.** Richest CRD taxonomy for in-cluster faults (PodChaos, NetworkChaos, HTTPChaos, DNSChaos, IOChaos, TimeChaos, StressChaos, KernelChaos, JVMChaos, AWSChaos, GCPChaos, AzureChaos, BlockChaos). The `Workflow` CRD composes experiments with dependencies, timeouts, and conditions. Native support for label-based selection.
- **Weaknesses.** Dashboard UX is engineer-targeted; non-engineers have a learning curve. KernelChaos requires a privileged daemon which some security postures do not permit.
- **AI fit.** The default choice for an in-cluster AI chaos program. HTTPChaos on the inference gateway's egress is the cleanest way to inject provider-timeout faults; `Schedule` + `Workflow` is the cleanest way to automate per-change chaos.

### 3.2 LitmusChaos

- **Shape.** A CNCF project, Kubernetes-native CRDs, ChaosHub for sharing experiment templates, ChaosCenter dashboard, deep Argo integration, GitOps-native.
- **Strengths.** Public chaos-experiment hub (hub.litmuschaos.io) is the deepest community-curated catalogue. GitOps-first design means every chaos run is traceable to a git commit, which the compliance-aware function values. Argo-native workflow composition.
- **Weaknesses.** More moving parts (ChaosCenter, multiple operators) than Chaos Mesh for a small installation.
- **AI fit.** Strong choice when the function already runs Argo Workflows or Argo CD for serving deploys. Experiments can be versioned alongside serving manifests. The hub's community templates are a useful cross-check against the AI-specific catalogue (chapter 03) — if a class is missing from the hub, it is a candidate for upstream contribution.

### 3.3 Gremlin

- **Shape.** A commercial SaaS product with agents (in-cluster and on VMs), a central control plane, an attacks catalogue, scheduled scenarios, a "reliability score" aggregation.
- **Strengths.** Mature non-Kubernetes coverage (bare-metal, VMs, Lambda via wrappers). Scenario composition for game-day shape. Audit logging and RBAC are production-grade.
- **Weaknesses.** SaaS trust / data-residency review required. Price. Some AI-specific experiments (GPU-pod-level emulation) still require composition on top.
- **AI fit.** Strong choice for a mixed-fleet function where some AI workloads run on raw EC2 / bare-metal GPU boxes (training rigs that are not in the service K8s cluster). Gremlin's agent-on-host model covers the serving K8s cluster *and* the training allocation; a Kubernetes-only tool does not.

### 3.4 AWS Fault Injection Service

- **Shape.** AWS-managed service, templated "experiments" composed of actions and targets, native integration with EC2, RDS, EKS, ECS, EBS, Lambda, Networking, and SSM. Stop-condition integration with CloudWatch alarms.
- **Strengths.** Infrastructure-level faults the in-cluster tools cannot do: AZ disruption, cross-region route-table disruption, EBS pause, API throttling simulations, Spot-instance reclamation. Managed, auditable, IAM-controlled.
- **Weaknesses.** AWS-only. Experiments outside AWS resources need another tool. No first-class AI primitives.
- **AI fit.** Essential for AWS-hosted fleets that depend on cross-AZ failover (`mod-113` is this module's downstream), Spot-reclamation drills, and region-wide drills. Pair with Chaos Mesh or Litmus for in-cluster faults; the two layers do not overlap.

## 4. Tool-to-catalogue map

The experiment catalogue in chapter 03 and the fallback-contract chaos in chapter 04 run across tools. The table below (repeated in the chaos-program policy) names the default tool per experiment class. Deviations are allowed with a written reason.

| Chapter 03 class | Default tool | Alternates |
|---|---|---|
| Serving-pod kill under in-flight stream | Chaos Mesh `PodChaos` | Litmus `pod-delete`, Gremlin Shutdown |
| GPU-host cordon / drain | Scripted `kubectl` under chaos-wrapper + FIS instance-stop | Litmus node-drain |
| Prompt-cache invalidation | HTTPChaos on cache egress or application API | Direct cache-key purge job |
| Provider-timeout injection | Chaos Mesh `HTTPChaos` / `NetworkChaos` on egress | LiteLLM / gateway synthetic fault |
| Degraded-mode routing drill | Breaker-override via router config + chaos trigger | Dedicated router-chaos operator |
| Agent tool-call fault | HTTPChaos on tool API + synthetic error-shape injector | Application-layer tool-mock |
| KV pressure / graceful preemption | StressChaos (memory-hog) on serving container | Synthetic long-prompt load generator |
| Judge-model unavailable | HTTPChaos on judge endpoint | Judge-pipeline circuit-open simulation |
| Retrieval-index drop | Vector-DB admin API + chaos-wrapper, or PodChaos on index pod | Index-replica failover drill |
| Cross-AZ / region failure | AWS FIS AZ-disrupt | Multi-region failover drill |

### 4.1 The egress-shaping pattern

Provider-timeout, judge-unavailable, embedding-service-slow, tool-API-fault — these experiments all share an injection point: **egress traffic from the service to a specific upstream**. Chaos Mesh's `HTTPChaos` and `NetworkChaos`, applied at the pod-selector level with a destination filter, are the canonical mechanism. A production-realistic pattern:

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: HTTPChaos
metadata:
  name: inject-anthropic-500-on-summariser
  namespace: inference
spec:
  mode: all
  selector:
    labelSelectors:
      app: summariser
      chaos.target: "true"
  target: Request
  port: 443
  method: POST
  path: /v1/messages
  request:
    body: ""
  abort: true     # or delay: 30s for a stall simulation
  duration: "5m"
```

The `chaos.target: "true"` label is set only on the pods that are in the experiment's blast radius — a 1% canary slice. The egress match on `/v1/messages` restricts to Anthropic's API; other upstreams stay healthy.

The equivalent for provider stall rather than abort uses `delay: 30s`, which is the AI-specific case: the primary stays "connected" but blocks, so the breaker has to fire on the stall, not on an error response. Testing the breaker in both shapes is a chapter 04 requirement.

### 4.2 The GPU-host-level chaos pattern

GPU-host chaos combines Kubernetes primitives with a chaos-wrapper that respects the function's blast-radius controls. A representative shape:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: chaos-cordon-gpu-node
  namespace: chaos
spec:
  template:
    spec:
      serviceAccountName: chaos-operator
      restartPolicy: Never
      containers:
        - name: cordon
          image: bitnami/kubectl:latest
          command:
            - /bin/sh
            - -c
            - |
              NODE=$(kubectl get nodes -l gpu.pool=a100-serving,chaos.target=true -o jsonpath='{.items[0].metadata.name}')
              kubectl cordon "$NODE"
              kubectl drain "$NODE" --ignore-daemonsets --delete-emptydir-data --timeout=5m
              echo "drained $NODE"
```

The wrapper runs under a chaos ServiceAccount restricted to cordon / drain on labelled nodes. The `chaos.target=true` node label is set manually at experiment start and removed after; the operator cannot select every GPU node by accident.

### 4.3 FIS for cross-AZ / cross-region

The in-cluster tools cannot sever an AZ cleanly. AWS FIS can. A representative experiment template:

```json
{
  "description": "Simulate AZ us-east-1a network isolation for inference fleet",
  "targets": {
    "inference-az-1a-subnet": {
      "resourceType": "aws:ec2:subnet",
      "resourceArns": ["arn:aws:ec2:us-east-1:...:subnet/subnet-abc"],
      "selectionMode": "ALL"
    }
  },
  "actions": {
    "az-disrupt": {
      "actionId": "aws:network:disrupt-connectivity",
      "parameters": { "duration": "PT10M", "scope": "all" },
      "targets": { "Subnets": "inference-az-1a-subnet" }
    }
  },
  "stopConditions": [
    { "source": "aws:cloudwatch:alarm", "value": "arn:aws:cloudwatch:...:alarm/slo-burn-5min" }
  ],
  "roleArn": "arn:aws:iam::...:role/FISExperimentRole"
}
```

The stop-condition wires the experiment to the SLO-burn alarm — the auto-halt from chapter 01 §4.1 — mechanically.

## 5. Installation and lifecycle hygiene

Each tool's docs are the source of truth for installation. Three cross-cutting hygiene notes an AI SRE should absorb before exercise 02:

- **Namespace the operator, RBAC the operator tightly.** Chaos operators are privileged by design — they kill pods, modify iptables, cordon nodes. A permissive install is a blast-radius hole. Scope the operator's ServiceAccount to the chaos namespace + target namespaces, and audit via `kubectl auth can-i`.
- **Version the experiments in a git repo, not in the dashboard.** Every chaos run should be traceable to a commit. The dashboards are for inspection, not for authoring. ChaosCenter's GitOps-first design encodes this; Chaos Mesh users achieve it by committing the CRDs to the serving-manifest repo.
- **Dry-run everywhere first.** Chaos Mesh's `dry-run` mode and FIS's "preview" template run the controls without injecting the fault. Every new experiment runs dry once, in staging once, and in a small production slice once, before it joins the automated cadence.

## 6. A substrate-readiness checklist

Before the function runs the catalogue in chapter 03, the Kubernetes substrate should pass these checks. Exercise 02 is the forcing function.

1. **Cluster meets prerequisites.** The chosen tool's version matrix is satisfied; privileged daemons are allowed where required, or alternate modes are configured.
2. **Target labels are canonical.** `app`, `workload`, `gpu.pool`, `chaos.target` labels are applied consistently; the chaos operator's selectors match.
3. **Observability is wired.** Chaos events post to the time-series store and the on-call channel; the chaos annotation appears on the serving dashboard's panels.
4. **Baseline is stored.** The three metric families from chapter 01 §2 have a 7-day baseline available for comparison.
5. **Auto-halt is live.** The chosen stop-condition mechanism (CloudWatch alarm → FIS stop-condition, Prometheus alert → chaos-controller halt webhook) is tested on a dry-run.
6. **On-call knows.** The runbook `rb-chaos-experiment-active` (new, named here, written in exercise 02) tells the on-call how to recognise and abort an active experiment, and how to read the chaos event stream in incident channels.

## Summary

- Chaos tools for Kubernetes express the same five fault classes with different vocabularies: pod / container, network, I/O, time, stress.
- No tool has a first-class "GPU chaos" primitive. The AI SRE composes GPU chaos from pod chaos, HTTPChaos against DCGM, memory-stress on serving containers, and scripted cordon / drain wrappers.
- Chaos Mesh is the default in-cluster tool. LitmusChaos is the GitOps-first alternative. Gremlin is the SaaS cross-fleet option. AWS FIS is the infrastructure-level layer for AWS fleets.
- Egress shaping (HTTPChaos / NetworkChaos on label-selected serving pods towards specific upstream hosts) is the cleanest injection point for provider-timeout, judge-unavailable, embedding-slow, and tool-API faults.
- Operator RBAC, git-versioned experiments, and dry-run-first discipline are non-negotiable hygiene.
- Chapter 03 turns these mechanisms into the AI-specific experiment catalogue; exercise 02 builds the substrate on which that catalogue runs.
