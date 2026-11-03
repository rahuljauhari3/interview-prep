# AI inference systems briefing — November 3, 2026

## Executive take

The most important new result since yesterday’s FlatCake briefing is **cluster efficiency**: evaluating an inference deployment by useful tokens per dollar and per joule at the cluster level, rather than ranking devices by standalone throughput.

[Introducing Cluster Efficiency to LLM Serving](https://arxiv.org/abs/2611.02975), submitted November 2, implements this idea on a 32-chip Tenstorrent Blackhole Galaxy and compares it with an eight-GPU RTX PRO 6000 Blackwell server. Across three dense models, two precisions, and 8K–256K contexts, the paper reports 1.07–2.24× higher tokens per dollar for the Galaxy under its three-year acquisition-and-electricity model.

The important result is not that one vendor beats another. It is the ablation: replacing the Galaxy’s native links with emulated TCP-class communication reduces throughput by 23.5–67.4% and eliminates every reported tokens-per-dollar win. Economical compute remains economical only if collective latency and bandwidth allow its memory and compute to behave like a pool.

For CPU inference, this strengthens a familiar EPYC lesson at cluster scale: core count, DRAM capacity, and memory bandwidth are potential resources; topology determines how much of them becomes usable serving capacity.

**Assessment:** a substantive provisioning and communication result with useful trace characterization and ablations. The central principle is portable; the reported economic ranking is highly sensitive to pricing, utilization, software maturity, and the choice of GPU baseline.

## 1. The unit of optimization moves from device to serving pool

A single inexpensive accelerator can be slower and less energy-efficient than a premium GPU while a tightly connected pool of those accelerators is more cost-efficient. The apparent contradiction comes from spreading fixed costs—especially model weights—across more aggregate memory and serving a larger batch.

The paper separates two concepts:

| Metric | Question answered | Main blind spot |
|---|---|---|
| Device efficiency | How fast or efficient is one accelerator? | Ignores whether several devices pool economically |
| Cluster efficiency | How many useful tokens does the purchased and powered pool deliver? | Depends on workload, utilization, prices, and fabric |

For a model replica, a simplified useful metric is:

[
E_{mathrm{cluster}}
=
rac{	ext{SLO-compliant output tokens}}
{C_{mathrm{hardware}}/L + C_{mathrm{energy}} + C_{mathrm{operations}}}
]

where (L) is the amortization period. The paper uses a three-year hardware-and-electricity denominator. In production, the numerator should be **SLO-compliant** tokens, not unconstrained peak throughput; otherwise a cheap system with unacceptable TTFT or TPOT can appear efficient.

This also explains why peak FLOPS per dollar is insufficient. A serving pool must simultaneously provide:

- prefill compute;
- decode memory bandwidth;
- capacity for weights and resident KV state;
- low-latency collectives at every tensor-parallel layer;
- enough scheduling and software efficiency to keep the hardware occupied.

## 2. The agent trace makes capacity a cluster-level resource

The paper analyzes 357,149 model invocations from 4,265 coding-agent sessions. Its median step adds 875 fresh input tokens to a cached prefix of 118,656 tokens and generates 214 output tokens. Cached prefixes therefore constitute 95.7% of the median context.

For a Llama-3.3-70B BF16 KV representation, the authors estimate:

| Statistic | Context retained | KV state per session |
|---|---:|---:|
| P50 | 124,024 tokens | 22.6 GB |
| P90 | 256,770 tokens | 73.9 GB |
| P99 | 822,889 tokens | 139.3 GB |

At the busiest trace point, 36 open sessions collectively retain 9.6 million tokens, corresponding to roughly 3.15 TB of KV state under that representation.

This connects directly to yesterday’s FlatCake result. FlatCake asks how DRAM and NVMe should hold oversized session state. The new paper asks how many devices should jointly provide memory capacity and bandwidth before a slower tier is required. Both reject device-local capacity as the correct planning boundary.

For CPU serving, the equivalent hierarchy is:

1. CCD and LLC locality;
2. socket-local DRAM;
3. remote-socket DRAM;
4. CXL or remote memory;
5. NVMe or distributed KV storage.

Each boundary adds capacity and a communication cost. Provisioning should choose the cheapest boundary that still meets the latency objective.

## 3. Fast links turn small devices into a usable pool

On Llama-3.1-8B at 32K context, scaling from one to eight Galaxy chips over native links produces:

- 11.3× more maximum active users because weight storage is amortized across the pool;
- 5.8× faster 32K prefill;
- 6.95× faster one-user decode steps;
- 5.9× faster eight-user decode steps;
- only 4.8–6.5% of profiled decode time in collectives.

With TCP-class communication emulated on the same chips, eight-chip scaling falls to:

- 2.2× for prefill;
- 2.5× for one-user decode;
- 2.36× for eight-user decode;
- 62.6% of the decode step in collectives.

The hardware capacity is unchanged. What disappears is the ability to use it before synchronization dominates.

The paper models collective time conventionally:

[
T_{mathrm{comm}}
approx
sum_{i=1}^{K}
left(
alpha_i + rac{V_i}{B_i}
ight)
]

For tensor-parallel decode, the latency term (alpha_i) is often decisive because each layer performs small or moderate collective operations. Adding bandwidth without reducing launch, synchronization, routing, and hop latency can leave low-batch decode almost unchanged.

### What the implementation contributes

The Galaxy serving path combines:

- integrated communication engines on each chip;
- pipelined ring transfers;
- overlap between communication and computation;
- collectives shaped around the physical links;
- device-side replay of decode work to reduce host-driven launch gaps;
- vLLM integration above the device runtime.

The transferable idea is that a fabric is not useful merely because it has a large advertised bandwidth. The runtime must make collectives follow the topology, pipeline transfers across hops, overlap them with useful kernels, and remove host/control-plane gaps from the token loop.

## 4. Implications for EPYC and CPU inference

For a multisocket or multinode CPU deployment, “more inexpensive cores and DRAM” is only valuable if remote access and collectives remain below the slack in the serving SLO.

A useful provisioning model is:

[
T_{mathrm{step}}
=
T_{mathrm{local compute}}
+
T_{mathrm{local memory}}
+
T_{mathrm{remote exposed}}
+
T_{mathrm{barrier}}
+
T_{mathrm{runtime}}
]

The goal is not to minimize total transferred bytes. It is to minimize **exposed** remote time while maximizing the memory and batch capacity obtained from additional sockets or nodes.

For an EPYC-based stack, evaluate these crossover points separately:

- one CCD group versus a full socket;
- one socket versus two sockets;
- replicated weights versus tensor-parallel weights;
- socket-local KV versus sharded KV;
- one node versus RDMA-connected nodes;
- CPU-only execution versus CPU/GPU cooperative execution.

The key question for each expansion is:

[
Delta 	ext{useful throughput}
>
Delta T_{mathrm{remote}}
+
Delta T_{mathrm{sync}}
+
Delta T_{mathrm{runtime}}
]

subject to p99 TTFT and TPOT targets.

### Concrete engineering actions

1. **Report performance by topology, not only core count.** Include CCDs, sockets, nodes, NUMA placement, link type, and effective collective latency.

2. **Treat weight amortization as a first-class benefit.** Sharding weights can free disproportionate DRAM for KV state, so maximum concurrency may grow faster than socket count. Account for the communication required to obtain that gain.

3. **Measure collective latency at serving message sizes.** Large-transfer bandwidth tests do not predict dozens of per-layer reductions at batch 1–8.

4. **Keep the token loop resident.** Use long-lived worker teams, preallocated command structures, fixed scratch buffers, and persistent communication progress so every decode step does not rebuild tasks and wake thread pools.

5. **Map rings or trees to physical NUMA and NIC topology.** Avoid a logical collective order that repeatedly crosses the same Infinity Fabric or PCIe bottleneck.

6. **Optimize for admitted concurrency under SLO.** A configuration that holds twice as many sessions but increases p99 TPOT beyond the limit has not doubled useful capacity.

## 5. A practical benchmark for CPU cluster efficiency

Compare configurations at the same model, precision, context distribution, and latency objective:

| Configuration | Purpose |
|---|---|
| One socket, replicated weights | Establish the locality ceiling |
| Two sockets, replicated weights | Measure capacity gain without model-parallel collectives |
| Two sockets, sharded weights/KV | Expose weight-amortization versus Infinity Fabric cost |
| Two nodes over RDMA | Measure scale-out collective and runtime overhead |
| Two independent replicas | Determine whether data parallelism beats pooling |

For each configuration, report:

- admitted concurrent sessions;
- SLO-compliant output tokens/s;
- TTFT and TPOT p50/p95/p99;
- local and remote DRAM bytes;
- collective count, message-size distribution, and exposed collective time;
- barrier and thread-pool wakeup time;
- model-weight and KV bytes per socket;
- watts at the wall, not only package power;
- three-year cost sensitivity at multiple utilization levels;
- failure-domain and operational cost.

The most revealing comparison is often **pooling versus replication**. If a model fits independently on each node, two data-parallel replicas avoid fine-grained collectives and may outperform one pooled replica. Pooling earns its complexity when weight and KV capacity, model size, or skewed workloads prevent efficient replication.

## 6. Evidence quality and caveats

### Genuinely important

- The paper uses a large coding-agent trace rather than a short synthetic prompt distribution.
- It evaluates three dense models, BF16 and 8-bit state, and contexts from 8K to 256K.
- The slow-link ablation holds chips and capacity constant, isolating communication as the cause of the economic reversal.
- A single Galaxy chip loses to a single RTX PRO 6000 on both tokens per dollar and tokens per joule; the paper does not hide this. The benefit appears only after pooling.
- The implementation spans the serving engine, runtime, collective kernels, and link firmware rather than presenting a peak-bandwidth microbenchmark.

### Treat cautiously

- One author is affiliated with Tenstorrent, and the evaluated economical system is Tenstorrent hardware. The measurements deserve reproduction by independent users.
- The GPU baseline is an eight-card PCIe RTX PRO 6000 server, not an NVLink-connected B200/H200 system. This is defensible for a cost comparison but does not establish superiority over all premium clusters.
- The two systems use different 8-bit formats, so numerical behavior is not identical.
- Hardware prices are point estimates and can change substantially through discounts, cloud rental, depreciation, financing, and supply.
- A three-year acquisition-plus-electricity model omits engineering labor, rack/network infrastructure, spare capacity, software compatibility, failure recovery, support contracts, and opportunity cost.
- Tokens per dollar at the fastest batch is not automatically production goodput. The paper should be extended with arrival traces and explicit TTFT/TPOT SLOs.
- TCP-class behavior is emulated on the Galaxy rather than measured across a real multihost deployment with congestion and failure.
- The study covers dense models. MoE all-to-all traffic, sparse attention, multimodal preprocessing, and prefill/decode disaggregation may change the fabric requirements.

## Secondary development: vLLM 0.33.0 is broad, but not one breakthrough

[vLLM 0.33.0](https://github.com/vllm-project/vllm/releases/tag/v0.33.0) was released November 2 with 927 commits. The significant operational changes include CUDA 13-only published artifacts, PyTorch 2.14, broader low-precision KV-cache support, Model Runner V2 becoming the ROCm default, pipeline parallelism with prefill context parallelism, faster startup through precompiled kernel installation and retained autotuning data, and least-loaded routing across multiple API workers.

For an EPYC/ROCm deployment, the Model Runner V2 default change and 4-bit ROCm KV-cache path are worth compatibility testing. However, the release is an aggregation of model enablement, kernels, features, and fixes. Its headline commit count is not evidence of a single architectural advance, and most quoted percentages come from individual pull-request benchmarks rather than one controlled end-to-end evaluation.

## Reading priority

1. **[Introducing Cluster Efficiency to LLM Serving](https://arxiv.org/abs/2611.02975)** — Read the TraceLab characterization (§3), communication model and design (§4), implementation (§5), and the fast-link versus TCP-class ablation (§6.3).
2. **[vLLM 0.33.0 release notes](https://github.com/vllm-project/vllm/releases/tag/v0.33.0)** — Use as an upgrade and compatibility checklist, especially for ROCm, KV formats, CUDA packaging, and Model Runner V2.
3. **[FlatCake](https://arxiv.org/abs/2610.38355)** — Revisit alongside the new paper: FlatCake optimizes where oversized KV state resides, while cluster efficiency asks whether pooling more primary memory can delay or avoid slower-tier access.

## Bottom line

Yesterday’s result showed that DRAM should not be a mandatory KV-cache waypoint when agentic working sets exceed it. Today’s paper moves one level outward: the useful memory and bandwidth boundary may be an entire tightly connected pool, not one device or socket.

For CPU inference, the actionable lesson is not “buy cheaper accelerators.” It is to make the fabric part of the inference cost model. Quantify how much additional model and KV capacity each socket or node contributes, then subtract exposed remote-memory, collective, synchronization, and runtime cost under the actual latency SLO.

A scale-out configuration is efficient only when its links and execution runtime convert purchased capacity into timely tokens. Otherwise, it is merely inexpensive hardware waiting at barriers.
