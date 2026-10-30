# AI inference systems briefing — October 30, 2026

## Executive take

The most consequential new engineering material since yesterday’s request-adaptive hybrid-inference briefing is vLLM’s [Mistral Large 4 implementation report](https://vllm.ai/blog/2026-10-29-mistral-large-4). It is framed as day-zero model support, but the substantive content is a full-stack case study in serving a one-trillion-parameter sparse model whose optimizations interact rather than compose independently.

The report covers cross-layer KV sharing, sparse MLA, decode and prefill context parallelism, DSpark speculative decoding, host-memory KV offload, load-time expert requantization, and a specialized persistent decode kernel on AMD MI350-series GPUs.

Three findings are especially transferable to CPU and hybrid inference:

1. **State sharing must be expressed as ownership, not accidental aliasing.** Nine layers consume one earlier layer’s latent KV cache; the allocator and offload tier must deduplicate the state across every memory level.
2. **Capacity optimizations can reduce latency only after crossing a working-set threshold.** Decode context parallelism raises KV capacity nearly eightfold on TP8 but slows each token by 16–28% when capacity is not binding.
3. **Low-concurrency inference is becoming an execution-graph problem.** AMD’s specialized path fuses attention, sparse indexing, routing, experts, and cross-GPU reductions into one persistent kernel per layer, while adding explicit timeout and fail-stop semantics.

**Assessment:** genuinely important engineering work with unusually concrete measurements and implementation details. The model and hardware are specialized, so the headline performance numbers are not portable; the design constraints are.

## 1. Cross-layer KV sharing becomes a serving-stack contract

Mistral Large 4 has 72 decoder layers. Nine NoPE layers—39, 43, and every fourth layer through 71—read the latent KV cache produced by layer 35 instead of writing independent caches. Only 63 layers own KV storage.

vLLM introduced an explicit target-owner relationship in its attention interfaces. Consumer layers skip cache writes, and the model runner resolves their cache specification to the owner layer. This matters because cross-layer sharing must survive more than device allocation:

| Subsystem | Required behavior |
|---|---|
| GPU KV allocator | Allocate pages only for owner layers |
| Attention backend | Resolve consumers to the owner’s physical pages |
| Prefix cache | Hash and reference one shared state without multiplying refcounts incorrectly |
| Host offload | Store the owner cache once |
| Hot working set | Permit consumer-specific hot buffers without copying persistent KV |
| Transfer layer | Move the owner state once and rebuild consumer bindings |
| Eviction | Retain state while any consumer or request still references it |

The HiSparse host-offload implementation demonstrates why this is important. The nine consumer layers bind to layer 35’s resident GPU view and host copy, while keeping their own indexer-dependent hot buffers and LRU state. Deduplicating the persistent cache holds 14.3% more tokens for the same host pool than materializing the shared layers.

This connects directly to BellowsKV. Once multiple layers share state, compression ratio alone is not enough. The runtime needs a first-class state graph:

[
	ext{logical consumer}
ightarrow
	ext{state owner}
ightarrow
	ext{physical tier and pages}
]

For a CPU backend, this should include NUMA placement. A consumer layer should resolve to the owner’s socket-local pages or make an explicit replicate-versus-remote-read decision; it should not silently allocate a private copy through a generic cache path.

## 2. Sparse attention makes host DRAM useful—but only with fine-grained retrieval

The model’s DSA indexer selects the top 2,048 keys per query. HiSparse keeps the full latent KV cache in pinned host memory, maintains a GPU-resident working set plus a per-request hot buffer, and copies selected rows on a miss.

On a four-GPU GB300 configuration with a 440 GiB shared host pool, the report gives:

- 1.97 million KV tokens with GPU memory alone;
- 8.77 million tokens admitted at startup with HiSparse;
- up to 11.25 million tokens in steady state;
- unchanged GSM8K-200 score;
- successful needle-in-a-haystack tests near 98K and 248K tokens.

This is a 4.5–5.7× capacity increase, not a claim that host memory is as fast as HBM. The mechanism works because sparse attention narrows each step’s useful KV working set.

For CPU inference, the direction reverses but the principle remains: large persistent state already lives in DRAM, while the active attention working set should be moved toward the executing cores. A two-tier EPYC design could use:

- socket-local DRAM for the active sparse working set;
- remote-socket or CXL memory for cold latent KV;
- per-request hot buffers aligned with the thread/CCD placement;
- indexer output to drive asynchronous promotion.

The useful cost model is:

[
T_{mathrm{sparse}}
=
T_{mathrm{index}}
+
T_{mathrm{metadata}}
+
T_{mathrm{local hits}}
+
T_{mathrm{cold fetch}}
+
T_{mathrm{attention}}
]

Sparse attention wins only if indexing and irregular retrieval cost less than scanning dense KV. On CPUs, top-k selection, gather locality, cache-line overfetch, and TLB behavior can erase the arithmetic saving. Evaluate bandwidth at the **selected-row granularity**, not only total retained bytes.

## 3. Context parallelism is a capacity tool with a measurable latency tax

MLA uses one latent KV head, so tensor parallelism replicates its cache across ranks. Decode context parallelism instead shards the latent cache along the token dimension.

On TP8, DCP8 increases reported FP8 KV capacity from 2.38 million to 18.88 million tokens and NVFP4 capacity from 3.60 million to 28.76 million tokens—about 7.9×.

The raw implementation initially added roughly 7 ms per decode step at a 60K context because of:

- LSE and top-k candidate all-gathers;
- reduce-scatters;
- stable top-k merging;
- query gathers;
- small masking kernels and launch gaps.

The optimized path replaces those operations with symmetric-memory all-to-all LSE combination, direct top-k exchange into peer inboxes, top-k compaction, and an empty-row neutralization kernel. DCP-specific overhead falls to 3.2 ms. In the reported TP8/DCP8 FP8 run, p50 TPOT improves from 17.82 to 12.50 ms and concurrency-64 output throughput rises from 139.1 to 204.0 tokens/s.

The caveat is more important than the optimized number: DCP8 remains 16–28% slower per token than plain TP8 when KV capacity is not limiting. At up to 128 concurrent requests with 8K–60K contexts, plain TP8’s pool still fits, so it wins. For roughly 100K-token agent sessions, DCP prevents allocation pressure: at 32 sessions, the report shows approximately 0.7-second TTFT with TP4/DCP4 versus 15 seconds after plain TP4 runs out of KV.

The deployment rule is therefore threshold-based:

[
	ext{enable sharding when}quad
M_{mathrm{working set}}
>
M_{mathrm{local capacity}}
-
M_{mathrm{headroom}}
]

not “enable the widest context parallelism available.”

On multisocket EPYC, the analogous choice is local replication versus NUMA sharding. Measure the point where remote reads and partial reductions cost less than preemption, eviction, or reduced concurrency. That point depends on context distribution, not maximum advertised context length.

## 4. Speculative decoding consumes state capacity and can undo a cache win

The built-in DSpark drafter uses three sliding-window MLA layers with a 2,048-token window and a separate BF16 KV group. It accepts roughly 2.9 draft tokens per step on the tested NVIDIA workloads and raises output throughput about 1.9–2.2×.

Under DCP, the target cache is sharded but the drafter’s small window is replicated on every rank. That sounds inexpensive—about 7 MB per request—but allocator block geometry creates a larger effect. At a 64K context, enabling DSpark reduces capacity to about 0.69× that of plain DCP8 because each draft block’s ID corresponds to larger target-page coverage.

This is an excellent reminder that logical state bytes are not the same as allocator capacity:

[
M_{mathrm{effective}}
=
M_{mathrm{payload}}
+
M_{mathrm{page rounding}}
+
M_{mathrm{block coupling}}
+
M_{mathrm{replication}}
+
M_{mathrm{metadata}}
]

For CPU serving, speculative decoding should be evaluated against concurrency and NUMA capacity, not just acceptance length and tokens/s. A replicated draft cache can be harmless at low concurrency but force target KV across sockets at high concurrency, producing a discontinuous p99 regression.

Track:

- accepted draft tokens per step;
- target and draft bytes per request;
- allocator fragmentation;
- page/block coupling between target and drafter;
- socket-local capacity after draft replication;
- p99 TPOT at the maximum SLO-compliant concurrency.

## 5. AMD’s persistent decode path treats the whole layer as one kernel

The most interesting low-level implementation is the MI350 latency-specialized backend. For pure decode steps of one, two, or four requests, each of 71 MoE layers executes as one persistent FlyDSL kernel launch.

The fused path includes:

- input normalization;
- query and latent-KV projections;
- SSMax and positional processing;
- DSA index projection, cache insertion, scoring, and exact top-2,048;
- sparse MLA;
- output projection;
- top-4 expert routing;
- shared and routed MXFP4 experts;
- both tensor-parallel all-reduces through peer memory.

At TP4, this lowers per-token decode latency by 1.5× at one request and 1.1× at four versus the standard path. Combined with DSpark, the report gives 2.9× lower per-token latency than standard non-speculative decode for a single request.

This is more than operator fusion. It removes the host from the per-operator launch and synchronization path, fixes the supported batch shapes, and makes peer communication part of the kernel’s execution protocol.

The CPU analogue is not one enormous function. It is a persistent, shape-specialized execution plan that avoids repeated framework dispatch, thread-pool wakeups, tensor metadata construction, and barriers. For batch 1–4 decode on EPYC, consider:

- one long-lived worker team per socket;
- prebound packed weights and scratch arenas;
- static operator graph for common decode shapes;
- fused normalization/projection/activation paths;
- in-place routing metadata;
- explicit local reductions before cross-socket reduction;
- no general-purpose executor transitions between hot operators.

The acceptance bar is end-to-end TPOT, not the sum of kernel benchmarks. A monolithic plan can lose if it expands instruction footprint, prevents better GEMM blocking, or holds too many cores during memory stalls.

## 6. Persistent collectives require fail-stop semantics

A kernel that waits on peer GPUs can hang the service or emit corrupt output if one participant fails. The AMD path gives every in-kernel wait a ten-second timeout. Timing out sets a sticky fault word on all ranks; a device-side assertion checks it before logits are produced. If set, the remaining step is abandoned and the engine stops, ensuring no token from the failed step reaches a client. The check reportedly costs about one microsecond per launch.

The path is also default-deny. Unsupported topology or features cause startup to fail with the full set of incompatibilities rather than silently falling back inside a collective region.

This is a genuinely important reliability design, consistent with the earlier vLLM MoRIIO memory-reclamation RFC:

> A control-plane timeout is not proof that data-plane activity stopped, and a partial distributed step must not publish output.

For CPU collectives, the same contract should cover socket workers, processes, and remote nodes:

1. every barrier or peer wait has a bounded failure path;
2. failure state becomes visible to all participants;
3. partial reductions cannot reach sampling or client output;
4. buffers remain held until every possible accessor is quiescent;
5. unsupported execution combinations are rejected before starting the distributed region.

Performance-specialized paths should be stricter about correctness, not less strict.

## 7. Load-time requantization is becoming a deployment primitive

On ROCm, vLLM can load FP8 expert weights and requantize them layer by layer to MXFP4. The original FP8 copy is freed after each layer, so peak memory is the final MXFP4 model plus one FP8 expert layer rather than two full checkpoints.

The report gives:

- checkpoint footprint reduction from about 1,061 GB to 578 GB;
- expert weights falling from roughly 125 to 69 GiB per GPU at TP8;
- 22–71% higher output throughput than FP8 experts over concurrency 1–64;
- sufficient memory for two TP4 replicas on one eight-GPU node.

For CPU deployment, the pattern is directly useful. Shipping one canonical checkpoint and transforming it during load can avoid maintaining many hardware-specific artifacts. But the transformation must be:

- layer-streaming to bound peak RSS;
- NUMA-aware so final packed weights are first-touched on their execution socket;
- cached with a key covering ISA, kernel version, quantization scheme, and model revision;
- validated against direct quantization from the higher-precision source;
- observable so fallback to the original format is not silent.

A CPU loader that requantizes and packs weights should write directly into the final socket-local packed format rather than create an intermediate tensor and repack it later.

## What is genuinely important versus promotional

**Genuinely important:**

- The report publishes concrete before/after results, implementation constraints, known failures, and upstream pull requests.
- Cross-layer KV sharing is integrated through allocation, execution, and host offload rather than treated as a model-only trick.
- DCP is presented honestly as a capacity/latency tradeoff, including cases where it loses.
- Speculative decoding’s hidden capacity cost is quantified.
- The AMD specialized path includes fail-stop behavior and rejects unsupported configurations.
- Load-time requantization reduces both persistent and peak memory through layerwise transformation.
- The work spans NVIDIA, AMD, and TPU backends, revealing which abstractions are portable and which remain hardware-specific.

**Treat cautiously:**

- Most headline measurements use GB300 or MI350-series accelerators and cannot be projected to CPU inference.
- Several results use fixed-length, greedy workloads with EOS ignored and prefix caching disabled.
- GSM8K and limited needle tests are useful smoke tests, not comprehensive correctness validation for every topology.
- The 1T model’s architecture is unusual; results for sparse MLA, cross-layer KV sharing, and built-in DSpark do not generalize to ordinary dense transformers.
- Day-zero enablement still includes features marked for upstreaming and a documented HiSparse/all-reduce crash requiring a backend override.
- Reported improvements come from different configurations and should not be multiplied together.

## Worth reading

1. **[Day 0 support for Mistral Large 4 on vLLM](https://vllm.ai/blog/2026-10-29-mistral-large-4)** — The primary engineering report; prioritize cross-layer KV sharing, DCP, DSpark+DCP, ROCm specialized decode, and HiSparse.
2. **[Cross-layer KV-sharing implementation PR](https://github.com/vllm-project/vllm/pull/64095)** — Track how owner/consumer cache relationships enter generic vLLM interfaces.
3. **[DCP all-to-all combine PR](https://github.com/vllm-project/vllm/pull/61285)** — Useful implementation reference for replacing gather/reduce sequences with a direct symmetric-memory exchange.
4. **[DSpark under DCP PR](https://github.com/vllm-project/vllm/pull/63796)** — Shows how target-cache sharding interacts with replicated sliding-window draft state.
5. **[Mistral Large 4 serving recipe](https://recipes.vllm.ai/)** — Reproduction configurations for colocated and disaggregated deployments.

## Bottom line

Yesterday’s briefing focused on request-dependent execution paths. Today’s report shows the next layer of complexity: modern models combine shared state, sparse retrieval, multiple parallelism dimensions, speculative state, tiered memory, quantized experts, and specialized execution graphs. Each optimization changes the resource assumptions of the others.

For CPU and EPYC inference, the strongest near-term lessons are:

- represent cache sharing with explicit ownership across every memory tier;
- enable NUMA sharding only after the local working set exceeds capacity;
- include draft-state replication and allocator geometry in concurrency planning;
- build persistent low-concurrency execution plans that remove framework dispatch;
- stream requantization directly into final socket-local packed weights;
- require fail-stop semantics before publishing output from distributed or persistent execution.

The winning system will not simply have the fastest kernels. It will maintain a correct, topology-aware state and execution plan while the model changes what is stored, where it lives, and which path consumes it.
