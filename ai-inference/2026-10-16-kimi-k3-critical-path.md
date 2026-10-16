## Inference systems update — October 16, 2026

Two developments clear today’s significance threshold. AMD’s Kimi K3 engineering report is the stronger full-stack result; a new vLLM scheduler investigation is narrower but directly relevant to host-CPU efficiency.

### 1. AMD’s Kimi K3 work shows cache capacity can outweigh nominal compute peak

**Importance: High for AMD inference engineering and heterogeneous memory design**

AMD and Embedded LLM published a detailed account of scaling Kimi K3 on eight Instinct MI355X GPUs. On the public AgentX workload, the submitted stack delivered approximately 7,567 total tokens/s/GPU at 50 generated tokens/s/user P90 interactivity, versus 6,083 for the compared NVIDIA HGX B300 submission—a reported 24.4% advantage.

That headline needs care: the submissions used different software revisions, cache-offload implementations, concurrency points and dates. The more valuable contribution is the breakdown of how state placement, duplicate MoE work, kernel traffic and available parallelism interact in a hybrid MLA–KDA model. genui{"citation":{"refs":["turn52view1","turn53view0"]}}

**Read:** [Scaling Kimi K3 on AMD Instinct MI355X](https://vllm.ai/blog/2026-10-16-kimi-k3-mi355x)

#### Cache architecture: divide history, then spill it coherently

Kimi K3 has 24 MLA layers whose compressed KV history grows with context and 69 KDA layers that maintain fixed-size recurrent state. Ordinary TP8 divides query heads but replicates the complete MLA history on every GPU.

Eight-way decode context parallelism instead partitions cached positions. For a 142K-token conversation, AMD estimates that MLA history falls from about 1.96 GB to 245 MB per GPU before allocator overhead. Queries are exchanged across ranks, each GPU attends to its local history shard and partial results are merged with their normalization terms.

When state still exceeds HBM, vLLM’s CPU-offload path places MLA history and KDA checkpoints in host DRAM. The key correctness and performance constraint is that every required state type must share a compatible resume point. If MLA blocks exist through token 3,072 but the latest KDA checkpoint is at 2,048, execution must restart at 2,048 and recompute 1,024 tokens. Saving the attention-block tail with each KDA checkpoint aligns the resume boundary and trades a host-to-device transfer for avoided prefill computation. genui{"citation":{"ref":"turn53view0"}}

For an EPYC host, “CPU offload capacity” is therefore an incomplete metric. The useful quantity is restorable synchronized state:

[
C_{	ext{effective}}
=
min!left(
C_{	ext{MLA at compatible boundaries}},
C_{	ext{KDA checkpoints}},
C_{	ext{transferable within SLO}}
ight)
]

A large DRAM cache can produce little benefit if its layer states cannot resume from the same token or if remote-NUMA placement makes restoration slower than recomputation.

#### Removing duplicate MoE work without adding a collective

The original ROCm TP8 path gave every rank the same latent input and replicated the routed-expert output weights, causing all eight GPUs to calculate the complete output projection. The optimized path assigns output coordinates to ranks, adds each rank’s routed contribution to its shared-expert partial and uses an existing all-reduce to assemble the final vector.

This raised output throughput by 3.05–4.91% in the cited TP8 tests. A later change loaded only the expert-projection rows each rank actually computes, freeing about 3.85 GiB per GPU across 92 MoE layers.

The second change is especially instructive. It produced less than 1% throughput improvement at concurrency 32 and 48, but 28.9% at concurrency 70, where prefix-cache hit rate reportedly rose from 74.8% to 80.1%. The saved weight memory mattered mostly because it crossed a cache-capacity threshold—not because the projection kernel became proportionally faster. genui{"citation":{"ref":"turn53view0"}}

This reinforces a useful serving model:

[
Delta	ext{throughput}
=
Delta	ext{compute efficiency}
+
Delta	ext{cache residency}
+
Delta	ext{admission capacity}
]

Kernel benchmarks alone miss the latter two terms. When evaluating a weight-layout or sharding optimization on CPU, measure whether the saved bytes allow more socket-local KV state or active sequences, particularly near concurrency cliffs.

#### Small copies can be expensive because of launch and control overhead

The ROCm matrix-multiplication dispatcher previously packed inputs before selecting a kernel. When the fallback path was chosen, the copy had no consumer. Kimi K3’s 69 KDA layers therefore issued 69 unnecessary copies per decode step.

Each copy moved only 1,792 bytes but consumed roughly 4.8 microseconds; removing them saved about 0.339 ms per speculative step in the reported profile. Bandwidth was not the problem—the fixed cost of launching work was.

The CPU analogue is common in inference runtimes: small tensor copies, allocator operations, thread-pool transitions and format checks may move negligible data while still dominating fine-grained decode. Profile operation count and dispatch latency, not only bytes/s.

#### Match parallel decomposition to the actual shape

Several improvements came from exposing more independent work rather than increasing arithmetic peak:

- Padding 12 MLA query heads per GPU to 16 enabled an AITER assembly path that partitioned long histories across workgroups. At 32K input tokens, mean TPOT fell from 43.8 to 23.6 ms; the 1K case was unchanged.
- A fused KDA prefill path summarized ordered recurrent updates, scanned group summaries for starting states and replayed groups in parallel, improving input throughput by 5.3–5.6% in the reported configuration.
- Reducing speculative-verification tiles from 32 value columns to four increased available workgroups from 48 to 384. A layer-level kernel test fell from 16.20 to 5.61 microseconds.

For ZenDNN or CPU kernels, the corresponding question is not simply whether AVX-512 lanes are full. It is whether the request’s actual heads, experts, tokens and batch size expose enough independent tiles to occupy CCDs without creating excessive reductions or remote-memory traffic.

#### Practical EPYC experiment

Reproduce the cache portion of this design as a four-tier state system:

1. accelerator-local cache;
2. host DRAM local to the accelerator’s PCIe root;
3. remote-socket DRAM;
4. recomputation.

For each hybrid-model resume point, record:

- synchronized MLA/KDA token boundary;
- bytes restored by layer and state type;
- host-to-device and remote-NUMA bandwidth;
- restoration time versus suffix recomputation;
- cache hit rate and useful-hit rate;
- TTFT, TPOT and completed tasks per socket;
- accelerator stalls caused by the host path.

Define a useful hit as one whose complete compatible state can be restored before recomputation would finish. This prevents a nominal cache-hit metric from hiding unusable or badly placed state.

#### Caveats on the AMD–NVIDIA comparison

The published cross-vendor result is informative but not controlled enough to establish an architectural winner:

- the AMD and NVIDIA submissions used different vLLM commits and submission dates;
- AMD used native CPU offload while NVIDIA used Mooncake;
- concurrency, DCP mode and speculative depth can vary between interpolated points;
- the cost comparison uses modeled GPU-hour values with some assumptions unpublished;
- the trace revision was not recorded;
- several per-optimization measurements were separate experiments and did not compose directly into the headline curve.

**Assessment:** The headline should be treated as a competitive stack result, not a silicon-isolation benchmark. The engineering writeup itself is substantive. Its strongest lesson is that model serving performance emerges from state compatibility, memory capacity, duplicate work and scheduler-visible parallelism—not peak matrix throughput alone.

---

### 2. Repeated prefix-cache walks expose a scheduler CPU tax

**Importance: Medium-high for synchronous and CPU-sensitive deployments; proposed optimization is not merged**

A new vLLM investigation found that a waiting request with a long cached prefix can re-run the entire prefix-cache lookup on every scheduler step when it cannot be admitted under KV pressure.

The walk probes the block-hash table once per cached block. If a request remains at the head of the waiting queue for many iterations, the scheduler repeatedly rediscovers the same hit. A proposed memo stores the prior hit and each block’s primary hash, revalidates until the first changed block and resumes the ordinary walk from that point.

In a CPU-only scheduler microbenchmark with an 8,192-token shared prefix:

- hash-table probes fell from 736,080 to 265,150, a 64% reduction;
- lookup time fell from 313.3 to 159.4 ms;
- a still-valid 2,048-block lookup fell from 956.5 to 75.8 microseconds;
- total scheduler time improved by roughly 7%, close to the test’s reported noise band. genui{"citation":{"ref":"turn52view0"}}

**Track:** [vLLM issue #61604](https://github.com/vllm-project/vllm/issues/61604)

#### End-to-end effect depends on scheduler overlap

The important follow-up separated synchronous and asynchronous scheduling.

On an RTX 3080 Ti with a 16K shared prefix and deliberate KV pressure, the memo improved synchronous output throughput from 258.50 to 282.56 tokens/s (+9.3%) and reduced median ITL from 20.44 to 18.48 ms. On a second RTX 4090/Xeon system, synchronous throughput improved from 409.89 to 457.52 tokens/s (+11.6%), median ITL fell 13.2% and median TTFT fell 10.5%.

Under vLLM’s default asynchronous scheduler, throughput changed by only 0.5–0.9%, within the reported run-to-run noise, because scheduler work overlapped accelerator execution. genui{"citation":{"ref":"turn52view0"}}

This is a clean example of why a large micro-optimization may disappear end to end:

[
Delta T_{	ext{service}}
approx
max(0,Delta T_{	ext{scheduler}}-T_{	ext{overlap slack}})
]

The optimization matters when scheduling is on the critical path. That includes pooling models, pipeline parallelism greater than one, executors without async support and speculative methods that do not use the supported asynchronous paths.

#### Why this matters more on CPU inference

For CPU-only serving, scheduler work and model execution compete for the same cores, LLC and memory controllers. “Asynchronous” does not make host overhead free; it can merely move the interference elsewhere.

A CPU evaluation should test three placements:

- scheduler sharing inference cores;
- scheduler pinned to reserved cores on the same NUMA node;
- scheduler isolated on another NUMA node.

Measure scheduler cycles per generated token, prefix-hash probes, LLC misses, cross-NUMA reads, runnable-queue delay and inference throughput. A remote scheduler may reduce core contention while increasing cache-metadata latency; the best placement will depend on whether weight/KV bandwidth or scheduler serialization is dominant.

#### Caveats

- The memoization patch has not been submitted or merged.
- The fast path currently targets a narrow unitary full-attention configuration.
- Default asynchronous GPU serving showed no measurable throughput gain.
- The evaluation uses synthetic prefix-repetition workloads designed to create long cached waits.
- Correctness was checked through identical scheduling-decision traces and output comparisons, but broader eviction, reset and multi-cache-group stress testing is still required.

**Assessment:** This is not yet a production change, but it is a credible and well-measured performance finding. It identifies a CPU-side algorithmic cost—repeated (O(P)) prefix walks for a blocked request—that can become visible once scheduling is synchronous or model steps are short.

### Bottom line

The AMD report and scheduler investigation point to the same conclusion from different layers: optimize the system around the *actual critical path*.

For Kimi K3, freeing a few GiB mattered most when it raised cache residency at high concurrency; tiny unused copies mattered because launch overhead dominated; padding nominally useless heads helped because it unlocked a better parallel schedule. In vLLM’s scheduler, a 64% reduction in hash probes mattered only when that work could not hide behind accelerator execution.

For CPU and EPYC inference, the highest-value next step is a unified critical-path profile that combines kernel time, scheduler work, synchronized-state compatibility, NUMA placement and admission pressure. That will expose whether the next improvement should be another microkernel, a cache-capacity change or the removal of repeated host-side work.