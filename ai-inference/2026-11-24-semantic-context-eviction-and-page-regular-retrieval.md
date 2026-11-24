# AI inference systems briefing — November 24, 2026

## Executive assessment

The strongest new work changes where long-context state reduction should happen.

[Contexture](https://arxiv.org/abs/2611.26980) operates above the attention kernel: it combines tool-execution semantics with runtime visibility into shared KV pressure and prefix residency, removing agent history only when the memory benefit justifies lost cache reuse and re-prefill interference.

[LongCat Recall Sparse Attention](https://arxiv.org/abs/2611.27233) operates inside sparse attention: it replaces a full-history retrieval scan with incrementally maintained, page-aligned indexes whose layout is co-designed with the selection kernels.

[MemOnce](https://arxiv.org/abs/2611.27973) changes the model architecture: it stores compact final-layer memory states once, retrieves them once per input, and reconstructs layer-specific KV on demand. Its reported memory reductions are dramatic, but it is not a drop-in optimization for existing models.

For CPU inference, these papers define three complementary optimization levels:

| Level | Main question | Representative work |
|---|---|---|
| Agent/runtime | Which history remains semantically necessary now? | Contexture |
| Sparse-attention system | How can useful history be located without scanning everything? | LongCat Recall Sparse Attention |
| Model architecture | Can history be stored once instead of once per layer? | MemOnce |

The practical lesson is to optimize context in that order: remove semantically superseded state, index the retained history with a hardware-friendly layout, and consider architecture changes only when model retraining or adaptation is acceptable.

## 1. Contexture coordinates semantic eviction with physical cache pressure

**Importance: High for agent serving and directly transferable to CPU runtimes**

[Contexture: Runtime Context Eviction for High-Throughput Agent Serving](https://arxiv.org/abs/2611.26980), submitted November 20, begins with a mismatch between the agent harness and the serving runtime.

The harness understands tool operations. It knows that a later file read may supersede an earlier read, that a successful write invalidates older file contents, or that a later command output replaces an earlier observation. The runtime, however, sees token sequences and KV blocks but not those semantic relationships.

Conversely, the runtime knows whether a prefix is still resident, how much shared KV pressure exists, and whether evicting context will force expensive re-prefill that interferes with ongoing decode. The harness lacks that global physical view.

Contexture combines both.

### Logical history versus execution context

The harness retains the full conversation history as the canonical logical record. Contexture constructs a smaller execution context for each model request without requiring the harness to discard its history or the model to change.

Tracing tokens to tool operations reveals that **64% of final-history tokens** in the recorded workloads are candidates for removal because later operations supersede them.

But removing every candidate immediately is not optimal. Editing an earlier span invalidates the unchanged prefix after that point, destroying reusable KV state. The next request must prefill the surviving suffix again, and that prefill can delay concurrent decoding.

Contexture therefore makes eviction conditional on both:

- semantic supersession;
- current cache pressure and prefix residency.

This is more principled than always compacting at the harness or always preserving prefixes at the runtime.

### Reported evidence

Across two models, two agent harnesses, and two benchmarks, Contexture reports:

- **20–73% higher throughput under SLO** than the best of no eviction, harness compaction, AgentDiet, and Continuum;
- **7–63% higher throughput** than observation masking in five of eight configurations;
- **23% higher throughput under SLO** than LMCache alone on H100;
- up to **11% lower GPU energy per completed session**;
- a mean live-agent accuracy change of **+1.4 percentage points**, with a worst-case loss of **3.7 points**.

Observation masking loses 3–28 points in the reported comparison, illustrating that hiding tool output from attention without a strong semantic policy can damage task performance.

### Why it matters for CPU and EPYC inference

On CPU, long agent histories consume both DRAM capacity and memory bandwidth. KV eviction is not only a capacity decision: shortening the attended sequence reduces per-token reads during decode. That can materially improve bandwidth-bound TPOT even when total DRAM capacity is sufficient.

A CPU implementation should expose two interfaces.

From the harness:

```text
span_id
tool_operation_id
resource_identity
supersedes_span_ids
must_retain
confidence / provenance
```

From the runtime:

```text
prefix_residency
local and remote bytes
expected re-prefill cost
current decode interference
NUMA ownership
memory-pressure tier
```

The eviction decision should compare future decode savings with immediate recomputation:

[
N_{	ext{future tokens}} 	imes C_{	ext{bytes avoided per decode}}
>
C_{	ext{re-prefill}} + C_{	ext{cache invalidation}}
]

This calculation should be per socket. Removing a span may be attractive when it eliminates remote-NUMA reads even if aggregate memory pressure is low.

### Engineering cautions

Semantic removal changes model input and is not lossless. Tool-output supersession is safer for resources with clear versioning—files, database rows, command results—than for reasoning or observations whose relevance is implicit.

A production policy should:

- preserve canonical full history outside the model context;
- retain identifiers and summaries that explain removed state;
- allow task- or tool-specific retention rules;
- measure success rate alongside throughput;
- fall back when supersession confidence is low;
- log every context transformation for debugging.

### Assessment

This is a substantive systems contribution because it connects information available at different layers of the agent stack and evaluates both performance and task quality. The strongest transferable idea is not its specific supersession rules; it is the joint optimization of semantic utility, cache residency, and shared-system interference.

**Read:** [Contexture: Runtime Context Eviction for High-Throughput Agent Serving](https://arxiv.org/abs/2611.26980)

## 2. LongCat Recall Sparse Attention makes retrieval itself sublinear and page-regular

**Importance: High for million-token sparse attention; indirect but technically relevant for CPU indexing**

[LongCat Recall Sparse Attention: Co-Designing Key Organization and Retrieval for Long-Context Inference](https://arxiv.org/abs/2611.27233), submitted November 20, identifies a growing bottleneck in retrieval-based sparse attention.

Selecting only a small Top-K set reduces attention computation, but systems such as DeepSeek Sparse Attention may still score the entire visible key history to find that set. At million-token contexts, key selection can dominate the sparse-attention operation.

LRSA changes both the index structure and its physical layout.

### Index design

The sequence is divided into time buckets that can be indexed independently. New keys extend the index without regrouping old history.

Within each bucket:

1. keys are assigned to capacity-balanced directional groups;
2. complete groups are packed into pages;
3. each group retains its own center;
4. centers nominate pages;
5. refinement scores members of nominated pages plus fixed candidates such as sink, sliding-window, and not-yet-indexed tokens.

Keeping multiple fine-grained representatives per page avoids the accuracy loss of one coarse page centroid, while packing complete groups makes member access regular.

### Systems co-design

The paper also addresses the costs that often make an elegant index lose end to end:

- reordered pages provide contiguous member access;
- construction stages are fused according to group size;
- fixed candidates shared across queries are stored once during chunked prefill;
- index construction for new buckets overlaps subsequent computation;
- compatible bucket indexes and member pages are reused after prefix-cache matches;
- the implementation is integrated into SGLang’s inference path.

At a 1,024K-token context, LRSA reports:

- **4.70× faster key selection** than DSA;
- **35.60× faster combined selection and attention** than full attention.

The second number uses full attention as the baseline and should not be read as a 35.6× gain over an optimized sparse-attention system.

### CPU relevance

The algorithm is GPU-oriented, but its layout principles apply strongly to CPU execution.

A CPU sparse-retrieval path should prefer:

- independently appendable buckets;
- page-sized or huge-page-aware storage units;
- balanced groups that avoid variable-length gather loops;
- centers stored compactly enough for LLC residency;
- contiguous refinement candidates;
- vectorized scoring over page members;
- NUMA-local indexes paired with their KV pages.

The wrong CPU port would maintain a pointer-heavy graph or cluster structure whose random accesses defeat hardware prefetch and cause remote-socket lookups. The right port separates a small, cache-resident representative scan from batched, contiguous page refinement.

Benchmark selection separately from attention. Record:

- center-scan time;
- pages selected per query;
- refinement bandwidth;
- LLC and TLB misses;
- remote-NUMA reads;
- index-update cost;
- recall against full Top-K;
- end-to-end TPOT and TTFT.

### Assessment and caveats

LRSA is a serious algorithm-system co-design result. It targets a real second-order bottleneck that appears after attention arithmetic has already been sparsified.

Its value depends on models trained or validated for this sparse-attention scheme. The abstract’s performance results do not establish quality portability to arbitrary pretrained models, and the longest-context headline may overstate benefits for common 8K–64K deployments. Reordered KV pages also complicate integration with generic cache tiering and eviction.

**Read:** [LongCat Recall Sparse Attention: Co-Designing Key Organization and Retrieval for Long-Context Inference](https://arxiv.org/abs/2611.27233)

## 3. MemOnce stores memory once and reconstructs per-layer KV on demand

**Importance: Strategically high; not a runtime-only optimization**

[MemOnce: A Scalable Memory Architecture for Large Language Models](https://arxiv.org/abs/2611.27973), submitted November 20, attacks the multiplicative structure of conventional KV storage.

A transformer stores token state for every attention layer. MemOnce instead:

1. retains a compact subset of final-layer hidden states in one global memory bank;
2. retrieves relevant memory states once per continual input;
3. reuses the selected indices across attention layers and decode steps;
4. projects those states into layer-specific KV only when needed.

The persistent memory bank can scale in CPU memory or storage, while active GPU state remains bounded.

The project reports that an adapted Qwen3.5-4B at 1M context achieves:

- **1,313× lower active memory-state cache**, reaching 25 MB;
- **6.9× lower prefill latency**;
- **3.5× higher throughput**;
- a reported **+5.77-point aggregate memory-performance result** over full-context Qwen3.5 on five memory benchmarks.

The accompanying [implementation](https://github.com/Clare-Nie/MemOnce) and [project page](https://clare-nie.github.io/MemOnce/) make the work more inspectable than an architecture-only proposal.

### Why this matters in practice

MemOnce moves computation from repeated historical attention into selective retrieval and on-demand projection. For CPU-backed memory tiers, this is attractive because the host stores one compact representation rather than layer-wise KV.

The key bandwidth comparison becomes:

[
B_{	ext{retrieved memory states}}
+
B_{	ext{KV reconstruction}}
quad 	ext{versus} quad
B_{	ext{layer-wise KV transfer}}
]

If the retrieved set is small and reused across layers, reconstructing KV near the compute device can be cheaper than transferring stored KV for every layer.

For a CPU-native design, the same principle could reduce DRAM traffic: store one compact memory representation and reconstruct layer-specific blocks into per-socket scratch space. Whether this wins depends on projection cost, retrieval quality, vectorization, and reuse across generated tokens.

### Why it is not yet a deployment recommendation

MemOnce requires architectural modules and adaptation or training. It cannot simply be enabled in an existing vLLM or ZenDNN deployment while preserving identical outputs.

The strongest reported results come from one adapted 4B model and long-context memory tasks. Important open questions include:

- scaling to much larger dense and MoE models;
- quality on ordinary instruction following and code generation;
- latency of retrieving from actual CPU or SSD tiers;
- interaction with batching and prefix sharing;
- update and deletion semantics for long-lived memory;
- whether reconstructed KV becomes a compute bottleneck.

Treat this as a model–systems direction worth following, not as a replacement for runtime KV optimization today.

**Read:** [MemOnce paper](https://arxiv.org/abs/2611.27973)  
**Code:** [Clare-Nie/MemOnce](https://github.com/Clare-Nie/MemOnce)  
**Project:** [MemOnce project page](https://clare-nie.github.io/MemOnce/)

## Secondary signal: edge–cloud speculative scheduling

[CGSpec](https://arxiv.org/abs/2611.29255) adapts speculative length from draft–target distribution divergence and schedules shared verification using estimated remaining work and heterogeneous client return times. Across reported 8- and 16-client configurations, it improves mean system throughput by roughly 10–12% and mean client throughput by 14–33% over compared methods.

The result reinforces a known deployment principle: speculative budgets and verifier scheduling must account for network and client heterogeneity. It is useful for edge–cloud systems but less important for CPU datacenter inference than the three items above, and its improvements are moderate.

## Recommended engineering actions

1. **Add semantic span metadata to agent traces.** Record resource identity, tool operation, and supersession relationships alongside token spans.
2. **Make context eviction runtime-aware.** Include prefix residency, NUMA locality, re-prefill cost, and concurrent decode pressure in the policy.
3. **Separate selection from attention in profiling.** Sparse attention can move the bottleneck into index lookup; report both components.
4. **Use page-regular sparse indexes.** Favor compact representatives and contiguous refinement pages over pointer-heavy structures.
5. **Co-locate index and KV ownership.** Keep representatives, member pages, and corresponding KV on the same NUMA node where possible.
6. **Evaluate compact-memory architectures with full cost accounting.** Include retrieval, transfer, KV reconstruction, adaptation quality, and batching—not only active cache size.
7. **Optimize at multiple levels in order.** Remove semantically obsolete context first, index retained history second, then consider model-architecture changes.

## Bottom line

The new work shifts long-context optimization from indiscriminate compression toward coordinated reduction.

Contexture removes state because later tool operations make it obsolete, but only when runtime cache conditions justify invalidating prefixes. LRSA retains history yet avoids scanning it through incrementally maintained, hardware-friendly pages. MemOnce questions why persistent history should be stored separately for every layer at all.

For an EPYC-oriented inference stack, Contexture and LRSA offer the most immediate design ideas: feed semantic provenance into a NUMA-aware cache manager, and organize sparse retrieval around cache-resident representatives plus contiguous local pages. MemOnce is the longer-term opportunity—co-designing the model so CPU or storage tiers hold one reusable memory representation instead of a multiplied layer-wise KV footprint.