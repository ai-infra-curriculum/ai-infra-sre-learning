# Planning for GPU-Generation Diversity in a Fleet

## Why this chapter exists

A capacity plan that assumes a single GPU generation will be out of date before the ink dries. In practice, a fleet that has been running for more than a year already mixes generations — the original Ampere pool nobody retired, the Hopper pool that is the current workhorse, the Blackwell / Hopper-refresh pool that arrived last quarter, plus the AMD MI300 pool somebody bought because the NVIDIA allocation did not land. The capacity planning problem is **not** "pick the one right GPU for next year." It is "operate a fleet that has three generations in it, forecast against a fourth landing in two quarters, and translate observability and SLOs across all of them without silently regressing."

This chapter is about that translation. It is explicitly **not** about picking kernels, tuning engines, or deriving per-generation TFLOPs from spec sheets — those are the `ai-infra-performance` peer's work (`mod-102` chapter on peer boundaries, and chapter 07 of this module). The AI SRE's job is to know (a) which generation changes are a **capacity event** vs. a **performance event**, (b) how to compare dashboards and SLOs across generations without lying to yourself, and (c) how to fail-over and rebalance a mixed fleet during a generation rollout or retirement.

## 1. A compact map of the generations you will encounter

The AI SRE does not need to memorise spec sheets, but does need a working mental map of what each generation is for, what it brought, and what it is likely to be the bottleneck on. Below is the map at the time of writing; the specific hardware available to your fleet depends on your cloud provider, your data centre vendor, your region, and your reservation cycle.

### 1.1 The NVIDIA line

- **Ampere (A100, A40).** The previous generation. Still ubiquitous for fine-tuning, embedding, and small-model inference. Not competitive on modern large-model inference against Hopper — HBM bandwidth and tensor-core throughput are both lower by a large margin. If your fleet still has Ampere nodes, they are there for a reason (reserved capacity, cost, specific low-utilisation workloads); do not assume they are interchangeable with Hopper for the decode-bound work.
- **Hopper (H100).** The default large-model inference GPU for the last two years. 80GB HBM3 at ~3.35 TB/s bandwidth on SXM, FP8 support (Transformer Engine), NVLink Switch System on DGX-H100. The `H100 SXM5 80GB` is the SKU most fleets have the most of; the `PCIe 80GB` variant exists at lower bandwidth and should not be assumed equivalent to SXM for decode-bound workloads.
- **Hopper refresh (H200).** Same compute as H100 at a higher memory specification: 141GB HBM3e at ~4.8 TB/s. The AI SRE's mental model is "same prefill throughput, meaningfully higher decode throughput, meaningfully larger KV-cache budget." See the authoritative per-model benchmark from the inference-engine peer for exact numbers; the qualitative shape is that H200 shifts the operating point for decode-bound and long-context workloads more than for prefill-bound ones.
- **Ada (L40S, L4).** Not a datacentre-inference card at the H100 class — Ada is primarily a workstation / media / inference-at-the-edge generation. L40S shows up in some inference deployments for small models and image / multimodal workloads. Capacity planning posture: do not pool L40S with H100 for the same large-model deployment; treat them as separate pools serving different models.
- **Blackwell (B200, B100, GB200, GB300).** The current generation. B200 brings a step change in per-GPU tensor-core throughput (multi-× over H100 for FP8 / FP4 workloads), 192GB HBM3e, and the NVLink-5 fabric. GB200 (Grace-Blackwell Superchip) pairs two B200s with a Grace CPU and NVLink-C2C; the GB200 NVL72 rack stitches 72 Blackwell GPUs into one coherent NVLink domain, which is a genuinely new capacity primitive — it enables tensor parallelism across 72 GPUs without the per-hop PCIe / Ethernet penalty. GB300 is the Blackwell refresh with higher memory and a slightly revised fabric. The Blackwell generation is **not** a drop-in replacement for Hopper at the fleet-operations layer: the fabric, the cooling (liquid for NVL72), the power envelope, and the engine-side kernel readiness all change.

### 1.2 The AMD line

- **MI300X.** AMD's current datacentre inference-and-training GPU. 192GB HBM3 at ~5.3 TB/s bandwidth, ROCm software stack, competitive on decode-bound memory-bandwidth-bound workloads for models that fit well on it. MI300A (APU variant for HPC) is less relevant to inference-first fleets.
- **MI325X, MI350X.** AMD's refresh and next-generation. Where available, they shift the comparison points without changing the core observation: a fleet that supports AMD exists on a different software stack (ROCm vs. CUDA) with a different engine support matrix.

### 1.3 Other accelerators

- **Google TPU (v4, v5e, v5p, Trillium / v6e).** Available only on GCP, with a different programming model (XLA / JAX, PyTorch-XLA, vLLM-on-TPU). Capacity planning on TPU looks similar in shape to GPU but different in detail: the compute primitive is a TPU chip paired with a TPU-interconnect pod, and the cross-generation observability translation is similarly non-trivial.
- **AWS Trainium / Inferentia.** AWS's own inference accelerators (Inferentia2, Trainium2). Separate software stack (Neuron SDK), separate engine support matrix. Capacity planning posture is the same as AMD or TPU: if your fleet uses them, they are a separate pool serving a defined subset of models with their own throughput curves.

### 1.4 What this chapter is not

This chapter does not list TFLOPs per generation, HBM bandwidth per SKU, or per-model throughput numbers. Those drift every release and are owned by the engine and performance peers; copying them into a curriculum file guarantees that someone reads an old number into a production forecast six months from now. The chapter teaches **how to reason across generations**, not **which numbers to memorise**. Where you need numbers, pull them from (a) the engine peer's current benchmark, (b) MLPerf Inference submissions for your model class, or (c) the cloud provider's published per-SKU documentation. All three move; your forecast must pull them fresh each cycle.

## 2. Capacity events vs. performance events

The single most important distinction a generation swap introduces:

- A **capacity event** changes *how much* a given replica can serve at the same operating point. More requests per second per replica at the same TTFT / TPOT SLOs. The fleet needs *fewer* nodes to meet the same demand (or can absorb more demand at the same node count).
- A **performance event** changes *how fast* a given request is served. TTFT or TPOT improves at the same throughput. The fleet is not serving more; it is serving the same requests *better*.

The two usually travel together but not in the same ratio. **This matters because the two drive different operational decisions**:

- A pure capacity event (e.g., migrating from H100 80GB to H200 141GB for the same decode-bound model that fits fine on H100) increases decode-batch size at the same per-step latency because KV-cache budget rises. The HPA scaling thresholds must be recomputed; the forecast's `max_num_seqs` per replica is now different; the autoscaler will keep too many replicas warm if left alone.
- A pure performance event (e.g., enabling FP8 Transformer Engine on H100 for prefill, same SKU, same fleet) improves TTFT at the same load. The TTFT SLO can tighten or the fleet can hold the same TTFT at higher throughput. The autoscaler's queue-depth target needs re-tuning but the node count does not immediately change.
- A composite event (e.g., H100 → B200) does both at the same time. Both the capacity model and the SLO operating point need recomputation. If the AI SRE treats it as "just a hardware upgrade," the fleet is operating against assumptions that stopped being true at the swap.

### 2.1 The checklist at the moment of a generation swap

When a generation swap is on the roadmap, the AI SRE produces, before the swap:

1. **Measured throughput curve** on the new generation for the exact model and engine: prefill tokens/s per replica, decode tokens/s per replica at a range of batch sizes, context lengths, and prefix-cache hit rates. Produced jointly with the engine peer using the load-test harness from chapter 06.
2. **KV-cache budget per replica** on the new generation: how many in-flight sequences at what average context length. This sets `max_num_seqs` and feeds Little's Law back into the forecast.
3. **Revised forecast** using the new throughput curves. Record what changed: GPU count required, operating utilisation, per-replica RPS.
4. **Revised SLO operating point.** If TTFT improves by 30% at the same load, do you tighten the SLO (and make the quality-engineering team's canary-eval tolerance tighter), hold the SLO (and bank the headroom), or hold the SLO and raise the per-replica throughput? This is a conversation; the capacity plan proposes an answer.
5. **Revised autoscaler thresholds.** HPA / Karpenter target queue depth, TTFT-based thresholds (chapter 05), and scale-down cooldowns all depend on per-replica throughput. All need recomputation.
6. **Revised failure-domain plan.** Blackwell NVL72 racks are a new failure-domain shape — losing one rack loses 72 GPUs, not 8. The failure-domain multiplier in the forecast must reflect it.

If any of these six pieces is missing, the swap is not ready to ship — not because the hardware is not ready, but because the operational envelope is undefined.

## 3. Cross-generation observability pitfalls

A mixed-generation fleet lies to its own dashboards unless the dashboards are built with generation as a first-class dimension. Four pitfalls recur.

### 3.1 The GPU-utilisation gauge is not comparable

`nvidia-smi`'s `utilization.gpu` field measures the fraction of time at least one kernel is running on the SMs. It says nothing about *what* the SMs are doing. On an H100 running a tiny decode batch, it reads 95%. On an H200 running the same tiny decode batch, it reads 95%. The H100 is at the edge of its memory bandwidth; the H200 is at 70% of its memory bandwidth because bandwidth went up and the batch did not. Comparing utilisation across generations as if it were comparable is a classic misread.

**What to do instead.** Report per-generation panels with per-generation signals:

- For decode-bound work, the useful signal is **HBM bandwidth utilisation** (DCGM counters `DCGM_FI_PROF_DRAM_ACTIVE` or equivalent). Comparable across generations because HBM bandwidth is the actual bottleneck.
- For prefill-bound work, the useful signal is **tensor-core utilisation** (DCGM `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`). Comparable across generations.
- For engine-side signals, prefer **effective throughput** (prefill-tokens/s, decode-tokens/s) at a measured operating point, not utilisation percent.

See `mod-104` chapter 03 for the DCGM foundations.

### 3.2 Per-token latency is not comparable without the context dimension

TTFT and TPOT depend on prompt length and decode context length. H200 wins more on long-context decode than on short-context decode because the win is HBM bandwidth and the KV-cache reads scale with context. A dashboard that reports `TTFT p95` with no context-length stratification will show a confusing picture during a mixed-generation rollout: H200 appears to have no advantage because the long-context traffic you care about is a small percentage of requests.

**What to do instead.** Stratify by context-length bucket on the TTFT / TPOT dashboards (0–512, 512–2K, 2K–8K, 8K–32K, 32K+). Report the comparison at the bucket level during rollouts. The chapter-01 workload-phase classification is enough to pick the right buckets for your workload.

### 3.3 Prefix-cache behaviour is generation-sensitive

Prefix cache hit rate is a pure engine-side concept, but **effective hit-rate throughput** depends on the generation's KV-cache memory and bandwidth. H200's 141GB lets the engine hold more prefix state; the effective hit rate on the same workload rises because the cache does not evict as aggressively. If you forecast on H100's measured hit rate and deploy on H200, the actual hit rate rises and the forecast overshoots.

**What to do instead.** When moving a workload between generations, re-measure prefix-cache hit rate after a stabilisation period. Treat the pre-swap number as a lower bound, not a prediction.

### 3.4 Failure-domain shape changes

The B200 NVL72 rack is a single logical failure domain containing 72 GPUs in one NVLink domain. A rack-level fault (power, interconnect, cooling) loses 72 GPUs. A fleet that assumes an 8-GPU node is the failure domain and runs 3-AZ placement is still suddenly vulnerable to a single rack. Similarly, GB200 racks sometimes have higher correlated-failure rates in the first months of a hardware generation (liquid cooling learning curve).

**What to do instead.** Explicitly recompute the failure-domain multiplier in the forecast (chapter 02 section 4) when the generation introduces a new failure-domain shape. Do not inherit the multiplier from the previous generation's fleet.

## 4. Translating between generations in practice

The AI SRE will run a *lot* of translation operations during a generation transition. Three canonical shapes.

### 4.1 Replica equivalence

Given a workload class and a per-generation throughput curve, how many H100 replicas equal one B200 replica? Answer: **do not report a single number**. Report a ratio per metric:

- Prefill-token-throughput ratio.
- Decode-token-throughput ratio at the operating batch size.
- KV-cache-budget ratio (sequences per replica).
- End-to-end TTFT / TPOT ratio at the operating point.

A "B200 is 2.5× an H100 for our workload" shorthand is fine for a quick budget conversation and dangerous for a capacity forecast, because the four numbers above are different and the dominant constraint for your workload is probably only one of them. The honest version: "B200 is 2.3× H100 on decode throughput, 3.1× on prefill throughput, 1.8× on KV-cache budget at FP8-weights, giving a composite capacity multiplier of ~2.1× for the chat workload and ~2.9× for the RAG workload."

### 4.2 SLO equivalence

When a generation swap improves TTFT, decide whether to tighten the SLO or hold it. Tightening is reasonable if (a) the new operating point is stable, (b) the tighter SLO improves user experience in a measurable way, and (c) the error-budget policy accommodates the headroom reduction. Holding is reasonable if (a) the fleet is heterogeneous and different generations hit the new target with different margins, (b) the product team is not asking for faster, and (c) the headroom is more useful as a shock absorber. The AI SRE proposes; the eval / product peers accept.

Hold the SLO if any part of the fleet still uses the old generation. The SLO is one contract across the fleet; stratifying it per generation turns the on-call's incident response into a "which GPU am I on today?" branching tree, which nobody wants.

### 4.3 Operational translation during rollout

During a rolling migration (replacing H100 replicas with H200 replicas over weeks):

- **The HPA threshold stays the same.** The threshold is the SLO-bound target queue depth; the fleet's per-replica throughput is rising, so the HPA will scale down naturally as H200 replicas take over.
- **The replica-count target decreases.** The forecast's node-count output drops; the quarterly review tracks it.
- **The prefix-cache metric must be watched.** As H200 holds more cache, hit rate rises; the forecast's sensitivity analysis must note this and the FinOps conversation may rebalance (cheaper per-token at higher hit rate).
- **The failure-domain plan is revalidated.** If H200 nodes are packed into denser racks, the AZ/rack/DC distribution may need rebalancing to hold the AZ-loss headroom.

The point is that a rolling migration is **not** operationally transparent. It is a multi-step forecast update in real time, which is why the capacity plan is a living document (chapter 02 section 6).

## 5. The special case: NVLink-72 / rack-scale coherent domains

GB200 NVL72 (and the upcoming GB300 variants) stitch 72 GPUs into one NVLink domain. Operationally this is a new capacity primitive. Three implications:

- **Tensor parallelism can span 72 GPUs** without the inter-node bandwidth penalty that previously capped TP at the node (8 GPUs over NVLink inside a DGX, degraded over InfiniBand across nodes). Very large models that previously needed pipeline parallelism across nodes can now be tensor-parallel across a rack.
- **The failure domain is the rack.** Losing an NVL72 rack loses 72 GPUs with a single blast. The fleet must be sized to survive it, which means at least two racks per workload if that workload is TP-72-wide.
- **Capacity reservation is at rack granularity.** Cloud providers price and reserve NVL72 at the rack level. The forecast's discrete step size is a rack, not a GPU. This changes the economics at small scale (one rack is a large commitment) and is comfortable at large scale (fewer, larger units to plan).

The AI SRE treats an NVL72 (or its equivalent on AMD / Trillium) as *one replica* for planning purposes when the workload is sized to use it that way, and the forecast's replica-count output drops by the TP degree. Do not confuse "72 GPUs" with "72 replicas"; the two are different numbers.

## 6. Deciding the generation mix for the next cycle

At each quarterly or annual capacity review, the AI SRE answers four questions about generation mix:

1. **What is the current fleet's generation distribution?** As-built, per-workload.
2. **Where is each workload bottlenecked?** Prefill, decode, KV-cache budget, HBM bandwidth. Per chapter 01.
3. **Which generation moves the bottleneck most for each workload?** H100 → H200 for decode-bound, H100 → B200 for prefill-bound, GB200 NVL72 for very-large-model workloads that need TP-across-rack, MI300X for fleets that have a strong reason to be on AMD and a tested software path.
4. **What is the supply and reservation reality?** The preferred SKU may not be available in your region at your scale on your timeline. The capacity plan must be *shippable*, which means the forecast's generation choice is constrained by the vendor's allocation calendar. Chapter 07 of `mod-107` addresses the vendor-side of this conversation.

The answer is a generation-mix recommendation like "hold the H100 baseline pool, migrate the long-context RAG surface to H200 by Q3 end, evaluate B200 NVL72 for the 70B-model-TP-72 experiment, do not add MI300X capacity this cycle." A one-line recommendation per generation, with the reasoning traceable back to measured throughput curves and the forecast.

## Summary

- **A fleet has multiple generations in it.** Operate the mixed fleet; do not pretend it is uniform.
- **Memorise the shape, not the numbers.** The map of Ampere / Hopper / Hopper-refresh / Ada / Blackwell / GB200 / GB300 / MI300 is what you carry. Numbers are pulled fresh each cycle from the engine peer, MLPerf, and the provider.
- **Separate capacity events from performance events.** A generation swap usually does both, in different ratios. The forecast and the SLO operating point each react to the right axis.
- **Run the six-piece swap checklist.** Throughput curve, KV-cache budget, revised forecast, revised SLO operating point, revised autoscaler thresholds, revised failure-domain plan.
- **Watch the four observability pitfalls.** GPU-util is not cross-generation comparable; TTFT needs context-length stratification; prefix-cache behaviour changes on generation change; failure-domain shape can change (NVL72).
- **Translate deliberately.** Replica-equivalence is a per-metric ratio, not a scalar. SLO equivalence is a product-level decision. Rolling migration is an active forecast update, not a passive swap.
- **NVL72 / rack-scale coherence is a new primitive.** Treat the rack as a replica when the workload is TP-across-rack; size for rack-level failure; plan reservations at rack granularity.
- **Produce a generation-mix recommendation per quarter.** One line per generation, traceable to measured throughput and the forecast.

The next chapter turns from own-fleet capacity to provider-side capacity — quota, rate limits, and reservation across the major model providers.
