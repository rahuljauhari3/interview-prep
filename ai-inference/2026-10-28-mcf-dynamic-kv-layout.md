# AI inference systems briefing — October 28, 2026

## Executive take

The main new result since yesterday’s BellowsKV briefing is **MCF**, a distributed long-context inference design that treats KV placement as a phase-changing runtime policy rather than a fixed sharding choice. BellowsKV reduced the amount of KV state retained across layers; MCF addresses the orthogonal problem of **where surviving KV blocks live and what moves over the network**.

The core idea is worth carrying into CPU clusters even though the prototype is FPGA-based: spend temporary memory early in prefill to replicate high-reuse prefix blocks, drain those replicas as capacity tightens, and enter decode with a compact owner-sharded layout. During decode, keep KV stationary and move only queries plus compact online-softmax state.

**Assessment:** substantive systems idea and a useful scheduling model; promising results, but not yet evidence of production readiness or direct portability to GPU/CPU servers.

## 1. MCF makes the KV layout change with the inference phase

**Importance: High for long-context scalability; medium-high for CPU inference**

[MCF: A Memory- and Communication-Aware Co-Designed Framework for Long-Context LLM Inference](https://arxiv.org/abs/2610.34731) was submitted October 26 and appeared in the latest research feed after the prior briefing.

Static context-parallel systems usually choose between two unattractive extremes:

- **Owner-only sharding:** memory-efficient, but later prefill chunks repeatedly fetch or circulate earlier KV blocks.
- **Replication:** improves locality, but consumes capacity and leaves an unsuitable layout for decode.

MCF’s Memory-Aware Replication and Scheduling mechanism, MARS, instead uses a lifecycle:

| Phase | KV policy | Intended effect |
|---|---|---|
| Early prefill | Replicate early, high-reuse KV blocks while memory headroom exists | Convert spare capacity into locality and communication overlap |
| Middle/late prefill | Reduce replica factors and mark copies for draining | Stay within the memory budget as the context grows |
| Prefill boundary | Retain only permanent owner shards | Avoid a bulk reshuffle before decode |
| Decode | Keep KV shards stationary; route queries and partial softmax state | Make communication scale with query/state size rather than accumulated context |

This is the paper’s strongest contribution. It recognizes that the value of a KV block is not static: an early prefix block has many future consumers, while a late block has little remaining prefill reuse. Replication should therefore be based on remaining reuse and available capacity, not merely on a global cache policy.

## 2. The result is about exposed latency, not minimum bytes transferred

MCF reports 16K-token prefill TTFT falling from 1.68 seconds on two logical nodes to 0.62 seconds on sixteen. At sixteen nodes, the paper reports 0.62 seconds for MARS versus 1.91 seconds for Ring Attention and 1.81 seconds for Zigzag Attention—about 3.1× and 2.9× faster, respectively.

The important qualification is that MCF transfers **more** data: 5.12 GiB at sixteen nodes versus roughly 4.40 GiB for both streaming baselines. Its win comes from pre-scheduling and overlapping communication, shortening paths to reused blocks, and moving work off the critical path—not from minimizing network volume.

That distinction matters in practice. A byte-count-only optimizer can reject a faster plan. The scheduling objective should look closer to:

[
min ; T_{mathrm{TTFT}}
quad 	ext{subject to} quad
M_{mathrm{owners}} + M_{mathrm{replicas}} + M_{mathrm{workspace}} le M_{mathrm{local}}
]

with the latency model including what communication remains exposed after overlap. For multisocket CPU inference, the analogous resource is not just NIC bandwidth; it includes remote-NUMA latency, socket-local DRAM bandwidth, memory-controller contention, and synchronization.

## 3. What transfers to EPYC and CPU inference

The FPGA compute-NIC is not directly relevant to a ZenDNN deployment, but four mechanisms are:

1. **Treat early-prefix replication as a temporary NUMA optimization.** Replicate the hottest initial KV blocks across sockets while per-socket memory has headroom, then retire replicas before decode reaches its steady-state memory limit.

2. **Separate ownership from residency.** Every block should have a permanent owner plus optional read-only replicas with explicit draining state. This avoids expensive migration when changing layouts.

3. **Choose chunk size from compute/communication overlap.** The paper finds larger chunks better in its tested range because they amortize scheduling, metadata, and movement overhead. On CPU, the optimum will also depend on LLC residency, OpenMP task overhead, AMX/AVX kernel efficiency, and remote-memory burst size.

4. **Keep decode KV stationary.** Route the current query to each shard, compute local attention statistics, then merge online-softmax state. For long contexts, moving a small query and reduction state is preferable to repeatedly moving the growing KV history.

A useful EPYC prototype would compare three policies at 8K–128K contexts:

- owner-only KV sharding;
- full per-socket replication until capacity failure;
- MARS-like temporary replication with progressive draining.

Measure TTFT, p50/p99 inter-token latency, local and remote DRAM bytes, Infinity Fabric traffic, replica bytes created, exposed copy time, and total copy time. The key test is whether extra traffic lowers critical-path latency without creating a later bandwidth cliff.

## 4. Evidence quality and caveats

The evaluation is narrower than the headline speedups suggest:

- The platform is a cluster of AMD-Xilinx KR260 boards, not commodity CPU or GPU servers.
- The model is Qwen2.5-1.5B with INT4 weights and KV entries.
- Reported prefill experiments extend to 16K tokens, modest by current long-context deployment standards.
- Baselines are dataflow-equivalent implementations on the authors’ substrate, not production vLLM, SGLang, or DeepSpeed deployments.
- Four-board decode reaches 17.3 tokens/s; the system is targeting a specialized low-power point, not datacenter throughput leadership.
- The paper acknowledges that short prompts may lose because replica creation and scheduling overhead cannot be amortized.

So the **policy insight is more portable than the measured speedups**. Before adopting it, reproduce the layout lifecycle on the intended fabric and workload distribution.

## Worth reading

1. **[MCF paper](https://arxiv.org/pdf/2610.34731)** — Read Sections 3–4 for the progressive layout and scheduling model, then Section 6.5 for the most revealing result: lower TTFT despite higher communication volume.
2. **[Ring Attention](https://arxiv.org/abs/2310.01889)** — Useful baseline for understanding why repeated KV circulation becomes expensive during chunked prefill.
3. **[Flash-Decoding](https://crfm.stanford.edu/2023/10/12/flashdecoding.html)** — Background for query-centric, KV-stationary decode and online-softmax reduction.

## Bottom line

Yesterday’s BellowsKV result asked, “How much KV state must be retained?” MCF asks, “Where should that state live at each phase, and which tensors should move?” The combined design direction is stronger than either alone: compress or eliminate redundant KV first, then dynamically place the remaining state according to reuse horizon, local capacity, and exposed communication latency.

For CPU inference, the immediate opportunity is a **NUMA-aware temporary-replication policy** integrated with chunked prefill—not another isolated microkernel. The success criterion is lower TTFT and stable p99 latency under long contexts, even if aggregate memory traffic rises modestly.
