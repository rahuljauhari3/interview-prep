# AI inference systems briefing — October 26, 2026

## At rack scale, the CPU control plane can erase faster GPU execution

**Importance: High for inference scalability and host-side performance engineering**

The most useful development since Friday’s Apex briefing is vLLM’s October 23 engineering report on serving DeepSeek-V4.1-Flash across a GB300 NVL72 rack. The headline—954K total tokens/s per GPU at a P90 of 196 tokens/s per user—is hardware- and benchmark-specific. The more transferable result is that wider expert parallelism shortened GPU execution but initially made end-to-end decode slower because CPU coordination could no longer remain hidden.

At decode expert parallelism 32, the unoptimized system took 17.57 ms per step versus 13.84 ms at EP16, even though EP32’s GPU work was 1.1 ms faster. As GPU steps became shorter and the synchronized rank count increased, host work moved onto the critical path and the probability of one lagging rank increased. genui{"citation":{"ref":"turn75view0"}}

**Read:** [Serving DeepSeek-V4.1-Flash at Scale](https://vllm.ai/blog/2026-10-23-deepseek-v41-flash-pd-wide-ep)

## The important fixes were control-plane algorithms, not new GEMMs

| Bottleneck | Change | Reported effect |
|---|---|---|
| Decode worker synchronously posts KV reads | Push KV with NIXL/UCX scatter-gather; send packed rows as one region | 141K → 440K tokens/s/GPU; P90 interactivity ~40 → 159 tok/s |
| Gloo CPU all-reduce grows linearly with ranks | Recursive-doubling CPU all-reduce | DP synchronization 10.8 → 1.5 ms; step time 16.10 → 13.15 ms |
| Admission bursts add up to 140 requests in one step | Cap new KV loads at four per step | 460K → 516K tokens/s/GPU; P90 109 → 153 tok/s |
| Admission work scales with prompt length | One int32 prompt buffer, skip null sliding-window blocks, chunked copy kernel | CPU admission time for an 855K prompt: 38.8 → 6.0 ms |
| Shared host tables use small GPU mappings | Allocate host-NUMA memory with 2 MB GPU pages | +2.8% prefill throughput at 32K new tokens |

The measurements come from different configurations and should not be multiplied together. They nevertheless identify four general scaling laws. genui{"citation":{"ref":"turn75view0"}}

### 1. Faster accelerators reduce the CPU-overlap budget

As device-side work shrinks, a fixed host cost consumes a larger fraction of each token step:

[
T_{mathrm{step}}
=
max!left(
T_{mathrm{device}},
T_{mathrm{host}}-T_{mathrm{overlap}}
ight)
+
T_{mathrm{sync}}
]

At small scale, asynchronous scheduling may hide most of (T_{mathrm{host}}). At wide EP, every layer synchronizes many ranks and the minimum slack across those ranks approaches zero. One slow scheduler, collective participant, allocator path, or Python-side copy can gate the entire instance.

This explains the apparently paradoxical EP32 result: reducing device execution by 1.1 ms did not help while host-side step time remained above it.

### 2. Collective complexity matters on CPUs even when tensor data is small

The DP synchronization fix reduced a CPU all-reduce from 10.8 to 1.5 ms by replacing a rank-linear pattern with recursive doubling. This was coordination metadata, not a large tensor collective.

The practical lesson for CPU and hybrid serving is to classify collectives by algorithmic message rounds as well as bytes:

[
T_{mathrm{collective}}
approx
alpha cdot R(P)
+
eta cdot B(P)
]

where (alpha) is per-round latency, (R(P)) is the number of synchronization rounds, (eta) is inverse bandwidth, and (B(P)) is transferred bytes. For tiny control messages, the (alpha R(P)) term dominates. A bandwidth-oriented optimization will miss the real cost.

For a multi-instance EPYC deployment, audit:

- request-count and load-balancing reductions;
- distributed scheduler barriers;
- health and progress exchanges;
- cache-allocation coordination;
- expert-load statistics;
- cancellation and completion broadcasts.

Any of these can become a rank-amplified serialization point even when its payload is only a few bytes.

### 3. Admission must be rate-limited like execution

Admitting up to 140 requests in one scheduler step caused a burst of KV loads and host work. Limiting admission to four new loads per step increased both throughput and interactivity in the cited rack configuration.

This is a useful correction to work-conserving scheduling. “Admit everything that fits” is not necessarily optimal when admission itself consumes:

- prompt serialization and copies;
- block-table construction;
- cache-transfer setup;
- allocator operations;
- page registration;
- routing and bookkeeping;
- CPU cache and memory bandwidth.

A better controller constrains admission by a host-time budget:

[
sum_{rin A_t}
widehat{T}_{mathrm{admit}}(r)
leq
B_{mathrm{host}}(t)
]

rather than only by available KV blocks. The budget can expand when decode steps are long and contract when wider parallelism shortens the GPU window.

For CPU-only serving, this is even more important because admission and inference compete for the same cores, LLC, and memory controllers. An admission burst can degrade active decodes without appearing as model execution time.

### 4. Virtual-memory geometry is part of inference performance

DeepSeek-V4.1-Flash uses shared Engram host tables with random GPU reads. Moving from a shared-memory mapping backed by 64 KB GPU pages to host-NUMA allocations that permit 2 MB GPU pages improved prefill throughput by another 2.8% at 32K new tokens. The authors attribute the change to fewer GPU TLB misses. genui{"citation":{"ref":"turn75view0"}}

For EPYC inference, the analogous checklist is broader:

- transparent huge pages versus explicit huge pages;
- first-touch placement of packed weights and KV arenas;
- TLB reach per CCD and socket;
- page migration and automatic NUMA balancing;
- shared-memory mappings used across processes;
- IOMMU mappings for accelerator-visible host buffers;
- page registration granularity for RDMA.

A host buffer can be physically local yet still perform poorly because its page size, registration, or mapping forces excessive translation work.

## The model architecture changed the optimal parallelism

The report also demonstrates why serving topology should follow the model rather than a general “scale wider” rule.

DeepSeek-V4.1-Flash has only 890 bytes of global KV per token because of cross-layer sharing and FP4 compression. At a 128K context, decode is still MoE-bound: the MoE path consumes 62% of the target step, while attention and indexing together consume 13%. Increasing context from 128K to 855K raises the measured target-forward latency by only about 7%. genui{"citation":{"ref":"turn75view0"}}

Those properties lead to different choices by phase:

- **Prefill:** a small DEP2 instance wins because larger DEP groups suffer more load imbalance. At concurrency 256, DEP4 achieved only 83% of DEP2’s per-GPU throughput.
- **Decode:** wide EP reduces dominant expert-weight reads, provided expert placement and host coordination are controlled.
- **Load balancing:** EPLB raises reported target-model expert balancedness from 0.35 to 0.94 and draft-model balancedness from 0.21 to 0.89.
- **KV capacity:** unlike many long-context models, KV is not the primary decode capacity limiter, so optimizing around cache scarcity would solve the wrong problem.

This is the practical design sequence:

1. measure the model’s bytes per token and per-layer time breakdown;
2. identify whether prefill is compute-, indexer-, or movement-bound;
3. identify whether decode is attention-, expert-, or control-plane-bound;
4. choose parallelism independently for the two phases;
5. remeasure end-to-end step time to expose host overhead and imbalance.

## Direct implications for ZenDNN and EPYC serving

For CPU inference, replace “GPU slack” with the time between global synchronization points. A useful profiler should place host activity and kernel activity on the same timeline and report:

[
H =
rac{T_{mathrm{host exposed}}}
{T_{mathrm{token step}}}
]

by concurrency, socket count, and parallel topology.

I would add the following regression sweep to a ZenDNN/vLLM CPU backend:

| Dimension | Values |
|---|---|
| Instances | 1, 2, 4, 8 per socket |
| Thread placement | compact CCD, spread CCD, cross-socket |
| Scheduler placement | shared cores, reserved local cores, remote socket |
| Admission cap | 1, 2, 4, 8, unlimited |
| Prompt length | 1K, 8K, 32K, 128K+ |
| Page policy | 4 KB, THP, explicit 2 MB huge pages |
| Collective | linear/ring, tree, recursive doubling |
| Workload | decode-only, prefill-only, mixed elephant/mouse |

Record:

- scheduler and admission CPU time per step;
- runnable delay and involuntary context switches;
- LLC and TLB misses per socket;
- local/remote DRAM traffic;
- barrier arrival skew;
- collective rounds and latency;
- primitive execution time;
- p50/p99 TTFT, TPOT, and worst streaming gap;
- completed requests meeting both latency SLOs.

The most important comparison is not the fastest isolated kernel. It is whether a topology with faster kernels leaves enough host headroom to sustain scheduling and synchronization at the target concurrency.

## What is genuinely important versus promotional

**Substantive:**

- The post isolates CPU overhead by comparing model-forward timing with serving-step timing.
- It publishes individual fixes with direct pull-request links and measured before/after effects.
- It shows a real scaling inversion where wider EP improves GPU time but worsens end-to-end time.
- It distinguishes phase-specific topology: narrow prefill, wide decode.
- It treats admission rate and control collectives as performance-critical algorithms.

**Treat cautiously:**

- The 954K tokens/s/GPU figure counts cached input, uncached input, and output tokens together; it is not comparable to ordinary output-token throughput.
- The 7.9× comparison is against an aggregated earlier single-node result, not identical hardware and topology.
- The cost figure uses DeepSeek’s published off-peak pricing rather than infrastructure ownership cost.
- Several optimization effects were measured in different setups and are not independently additive.
- The conclusions about KV capacity and context-length sensitivity are highly model-specific.

## Secondary reading: context parallelism for million-token serving

The companion [GLM-5.3 optimization report](https://vllm.ai/blog/2026-10-23-glm53-part2-pcp-dcp-hisparse) is also worth reading for its clean separation of query and cache sharding.

Prefill Context Parallelism partitions prompt queries by tokens rather than heads, eliminating replicated indexer work and increasing sparse-attention reuse. Decode Context Parallelism separately shards persistent KV by token. For a one-million-token GLM-5.3 request, the reported FP8 KV requirement is 53.4 GiB; DCP8 reduces persistent storage to about 6.7 GiB per GPU, at the cost of rebuilding a temporary full-context view through an all-gather each layer. genui{"citation":{"ref":"turn75view1"}}

The CPU analogue is to distinguish ownership from execution:

- persistent KV can be partitioned by NUMA node;
- queries can be scheduled to the nodes with the relevant work;
- temporary remote reads or reductions can reconstruct a logical full-context operation;
- the placement policy should compare persistent-capacity savings against repeated fabric traffic.

The report’s key caveat is explicit: DCP8 adds about 6% per prefill step at 64K cached context and about 15% at 248K. Capacity expansion is not free; it exchanges persistent replication for recurring communication. genui{"citation":{"ref":"turn75view1"}}

## Bottom line

The strongest lesson is that accelerator scaling increases the importance of CPU systems work. Once device steps become short, linear control collectives, synchronous KV operations, admission bursts, prompt-length-dependent copies, TLB pressure, and one lagging rank can consume the gain from wider parallelism.

For EPYC and ZenDNN deployments, the immediate action is to put the scheduler, allocator, collectives, page mappings, and NUMA placement on the same critical-path profile as the kernels. The next performance win may be a tree reduction, an admission budget, or a huge-page-backed arena—not another GEMM optimization.
