# AI inference systems briefing — October 29, 2026

## Executive take

The important new development since yesterday’s MCF briefing is **request-adaptive hybrid execution**: choosing, per request and per model block, whether to execute attention or a state-space-model (SSM) branch.

The new paper [Request-Adaptive Execution for Efficient Hybrid LLM Inference](https://arxiv.org/abs/2610.37432) describes a prototype called **Flex** that makes this model-level decision during prefill, freezes it for the request’s decode lifetime, and adds runtime support for divergent execution paths. The paper reports up to 3.87× higher throughput than its vLLM baseline on one H100.

The headline speedup is less important than the systems consequence: once execution becomes request-dependent, **a textual prefix is no longer enough to identify reusable state**, and a conventional batch is no longer one homogeneous operator stream. The serving engine must track execution provenance, form branch-specific sub-batches, and coordinate asymmetric branches without allowing split/merge overhead or tail imbalance to dominate.

**Assessment:** a genuinely interesting architecture/runtime co-design and a strong preview of where hybrid-model serving is heading. It is not yet a deployable optimization for existing models, and its GPU-only, custom-model evaluation does not establish the reported gains for CPU inference.

## 1. What changed: hybrid layouts move from model-static to request-static

Existing hybrid models such as Jamba and Nemotron-H fix their attention-to-SSM layout when the model is built. Every request executes the same branch at each layer, regardless of whether the task needs precise long-range retrieval or can tolerate a compressed recurrent state.

Flex pairs an attention branch and an SSM branch inside each hybrid block. During prefill, a lightweight gate uses the request’s hidden states to choose one branch per block. Those choices remain fixed throughout decoding.

That granularity is the paper’s most defensible design decision:

| Gating policy | Potential efficiency | State-management problem |
|---|---:|---|
| Static model layout | Low adaptivity | Simple; all requests share one execution graph |
| Token-level routing | Highest theoretical adaptivity | A later token may require KV or SSM state that previous tokens never produced |
| Request-level, fixed after prefill | Moderate/high adaptivity | One stable state type per request and block throughout decode |

Token-level switching would force recomputation whenever a request changes branch because attention requires historical KV while the SSM branch requires its recurrent state. Freezing the route after prefill preserves causal state continuity and removes gating from the token-by-token decode path.

This differs from yesterday’s MCF result. MCF adapts the **placement** of a fixed model’s KV state over time. Flex adapts the **computation and state type** used by each request, then makes the serving stack respect that choice.

## 2. Divergent execution turns batching into split, run, and merge

A conventional continuous batch assumes that every request traverses the same operator sequence. Flex breaks that assumption: at a given block, some requests execute attention and others execute SSM.

Its selective batching engine therefore:

1. keeps branch-independent work—normalization, shared FFNs, and related operations—in the original packed batch;
2. compacts requests into attention and SSM sub-batches only around the divergent branch;
3. executes the branches concurrently;
4. merges their outputs back into the original packed layout.

The synchronization boundary makes the layer latency approximately:

[
T_{mathrm{block}}
=
T_{mathrm{split}}
+
max!left(
T_{mathrm{attention}}(B_a,L),
T_{mathrm{SSM}}(B_s)
ight)
+
T_{mathrm{merge}}
+
T_{mathrm{shared}}
]

where (B_a) and (B_s) are branch-specific batch sizes and (L) is the active attention context length.

The paper uses profiled cost models and NVIDIA GreenContext to divide SMs between the branches. Reported prediction error is within 6.32% for attention and 5.18% for SSM. Against an ablation that splits batches without load-aware resource coordination, the full system reduces end-to-end latency by 21.7–28.2% on average across its three model scales.

The transferable lesson is that dynamic routing is not useful if one branch becomes the barrier tail. The optimization target is branch completion skew, not average utilization:

[
S_{mathrm{branch}}
=
rac{|T_{mathrm{attention}}-T_{mathrm{SSM}}|}
{max(T_{mathrm{attention}},T_{mathrm{SSM}})}
]

For CPU inference, an analogous runtime should assign threads, CCDs, and memory bandwidth according to predicted branch cost rather than splitting cores evenly. Attention becomes increasingly memory-bound with context length; the SSM path has a smaller persistent state and a different mix of recurrent kernels. Static thread partitions will frequently leave one pool idle while the other gates progress.

## 3. Prefix identity must include execution provenance

The paper’s most broadly important serving result is its cache-correctness observation.

Two requests can share identical prefix text but choose different branches in later hybrid blocks. One cached prefix may contain attention KV at a block where the new request requires SSM state, or vice versa. Reusing state based only on token hashes would be incorrect; rejecting the entire prefix would discard valid state from compatible earlier blocks.

Flex uses a two-level radix structure:

- an outer tree matches textual prefixes;
- an inner tree records the layer-selection path and its associated KV or SSM state.

Lookup first finds the longest token prefix, then reuses only the layer states compatible with the new request’s route. Eviction also operates at layer-state granularity rather than deleting every state attached to a prefix.

On the paper’s PubMed shared-context workload, this raises average effective hit rate from 18.7% to 43.4% and reduces average TTFT from 10.68 seconds to 5.92 seconds. At twenty competing articles, hit rate is 24% versus 7% for the conventional all-or-nothing prefix policy.

This principle extends beyond attention/SSM hybrids. A reusable-state key increasingly needs to encode:

[
K_{mathrm{cache}}
=
H(
	ext{tokens},
	ext{model version},
	ext{layer route},
	ext{precision},
	ext{positional state},
	ext{compression policy}
)
]

BellowsKV projections, sparse-attention layouts, early-exit routes, adapter selection, speculative modes, and mixed exact/approximate execution can all make identical tokens produce incompatible internal state. Prefix caches that hash only tokens and model identity will become either unsafe or unnecessarily conservative.

## 4. Reported performance and what it actually demonstrates

The evaluation uses one NVIDIA H100 SXM, an Intel Xeon Platinum 8458P host, and custom 4.4B, 14.5B, and 61B hybrid models. The 61B model uses FP8 to fit in memory. The runtime extends a vLLM-style loop and is compared with vLLM, SGLang, and Jenga using the same model and workloads.

The paper reports:

- up to 3.79×, 3.87×, and 3.44× vLLM throughput for the 4.4B, 14.5B, and 61B models;
- at the largest tested batch size, average TTFT reductions of 80.9%, 78.9%, and 75.2% versus vLLM, SGLang, and Jenga;
- average TPOT reductions of 71.6%, 63.1%, and 56.5% against those systems;
- the best result on 13 of 18 quality metrics across six reasoning, QA, and summarization datasets;
- up to 67.2% lower estimated TFLOPs per token than an attention-only Llama baseline.

These are substantial internal results, but they do **not** show that replacing an existing model’s static layers with runtime routing automatically yields a 3–4× production gain. The comparison includes a model-architecture advantage, a trained gate, selective batching, and specialized caching together. The strongest conclusion is that the co-designed stack can preserve batching and prefix reuse after introducing request-dependent execution.

## 5. Direct implications for CPU and EPYC serving

A CPU implementation would face a different resource-allocation problem than GreenContext. Threads are not interchangeable SMs: CCD locality, LLC capacity, first-touch placement, packed-weight ownership, and memory-controller pressure all matter.

A practical topology-aware executor should model each branch as:

[
T_b =
T_{mathrm{queue},b}
+
T_{mathrm{repack},b}
+
T_{mathrm{compute},b}
+
T_{mathrm{remote memory},b}
+
T_{mathrm{merge},b}
]

and choose both the branch sub-batch and its placement. The runtime should avoid moving requests between CCD pools if doing so loses packed weights or turns socket-local KV into remote reads.

For a ZenDNN/oneDNN prototype, the useful experiments are:

- compact branch indices once, then keep shared FFN tensors in the original packed layout;
- pin attention and SSM worker pools to separate CCD groups and allocate their persistent state locally;
- sweep dynamic versus fixed thread allocation by context length and branch ratio;
- cache packed branch weights per socket;
- measure whether sub-batch compaction and scatter/merge traffic remain in LLC;
- key prefix state by route signature and model artifact version;
- test partial-prefix reuse when routes diverge at an intermediate block;
- compare work-stealing against explicit branch reservations under bursty route distributions.

Record:

- split, compact, merge, and barrier time per block;
- branch completion skew;
- local and remote DRAM bytes;
- LLC and TLB misses;
- packed-weight reuse and reorder time;
- route-signature distribution and cache fragmentation;
- prefix hit rate at token, layer-state, and fully compatible levels;
- TTFT and TPOT p50/p99;
- throughput at fixed latency SLOs.

The CPU-specific opportunity is meaningful: if request-adaptive routing removes enough attention work and KV traffic, it may shift decode from bandwidth-bound toward compute-bound SSM execution. But small branch sub-batches can also destroy GEMM efficiency, and duplicated attention/SSM weights increase capacity pressure. Whole-model measurements are essential.

## 6. Important limitations and credibility checks

**Substantive:**

- The paper identifies two real runtime failures caused by dynamic model paths: batch fragmentation and unsafe/coarse prefix reuse.
- It freezes routes after prefill, giving a clear state-consistency contract.
- Selective batching preserves large shared operations while isolating only divergent work.
- Ablations separate the gains from branch coordination and hybrid-aware context reuse.
- Evaluation includes three model scales and six task datasets rather than one synthetic kernel benchmark.

**Still preliminary:**

- The arXiv metadata calls the system “Morph,” while the uploaded PDF consistently calls it “Flex.” This looks like a naming/versioning inconsistency and should be resolved before treating the artifact as polished.
- No implementation repository is linked on the arXiv page.
- The models require construction and fine-tuning from paired attention and SSM components; this is not a drop-in serving optimization for existing checkpoints.
- Evaluation uses one H100 server and controlled batches, not multi-node or production arrival traces.
- CPU performance is not evaluated despite the CPU host being listed.
- NVIDIA GreenContext is central to concurrent branch resource partitioning; equivalent control is not universally available across accelerators.
- The paper reports aggregate task metrics, but route stability and worst-case quality regressions for individual request classes need deeper analysis.
- Maintaining two candidate branches per block increases model-weight footprint even though only one branch executes per request.

## Worth reading

1. **[Request-Adaptive Execution for Efficient Hybrid LLM Inference](https://arxiv.org/pdf/2610.37432)** — Prioritize Sections 4–6 for the gating contract, selective batching, and cache-provenance design; Sections 8.3–8.4 contain the useful systems results and ablations.
2. **[Jenga: Effective Memory Management for Serving LLM with Heterogeneity](https://arxiv.org/abs/2503.18292)** — Background for managing mixed KV and recurrent-state caches.
3. **[Mamba-2: Transformers are SSMs](https://arxiv.org/abs/2405.21060)** — Useful for understanding the execution and state representation behind the SSM branch.

## Bottom line

Yesterday’s MCF briefing showed that KV placement should change across inference phases. Today’s result goes one level higher: the **operator path and state representation can change by request**, which forces the serving engine to make batching and cache identity path-aware.

For CPU inference, the immediate takeaway is not to reproduce the reported GPU speedup. It is to prepare runtimes for heterogeneous request paths: execution provenance in cache keys, branch-specific compact batches, topology-aware resource allocation, and direct measurement of split/merge overhead and branch-tail skew. Those mechanisms will matter for hybrid attention/SSM models, sparse MoE, early exit, adaptive precision, and other forms of conditional inference.
