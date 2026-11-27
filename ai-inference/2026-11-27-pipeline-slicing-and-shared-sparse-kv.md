# AI inference systems briefing — November 27, 2026

## Executive view

Three substantive results appeared after the November 26 briefing:

1. **PipeAxes decouples three decisions that chunked pipeline serving usually binds together:** prefill compute width, execution preemption granularity, and activation-transfer granularity. This is the strongest new systems paper for heterogeneous and bandwidth-constrained clusters.
2. **DFly2 removes the speculative drafter’s private KV cache and indexer** by reusing both the target model’s sparse KV and the target’s already-computed position selection. This is an important long-context memory and concurrency result, although current deployment support is incomplete.
3. **Suffix KV reuse makes in-place context editing cheaper.** A prototype for model-managed context reuses unchanged tokens after an edit, reducing prefill work by 4–15% per request. The result is smaller, but it points to a serving primitive that agent workloads will increasingly need.

| Development | Technical significance | CPU / heterogeneous-system relevance | Readiness |
|---|---|---:|---:|
| PipeAxes | High: new scheduling and transport decomposition | High | Research prototype |
| DFly2 | High: eliminates duplicate draft state and selection work | High conceptually | Weights/training code released; serving integration incomplete |
| Suffix KV reuse | Medium-high: enables efficient non-append-only contexts | Medium-high | SGLang prototype |

## 1. PipeAxes: pipeline serving needs more than one chunk-size knob

**Importance: High**

[PipeAxes: Decoupling Compute and Communication Granularities for Pipeline-Parallel LLM Serving](https://arxiv.org/abs/2611.34615), submitted November 25 and announced November 26, targets long-context serving across PCIe-connected or networked pipeline stages.

Conventional chunked prefill uses one token-chunk size for several unrelated purposes:

- GPU batch width;
- how long prefill runs before decode can resume;
- activation-transfer size between stages.

That coupling creates an unavoidable-looking tradeoff. Small chunks improve interactivity but repeatedly reload weights and underutilize compute. Large chunks improve compute efficiency but monopolize a stage and create large blocking transfers.

PipeAxes separates the axes:

- **Wide prefill batches** preserve efficient weight reuse.
- **Layer slicing** runs only a few layers before yielding, allowing decode work between slices.
- **Independent row-group transfers** split activations into smaller frames.
- **Hadamard-INT4 transport compression** reduces bytes crossing the link.
- **Slack-aware scheduling** independently selects batch width, slice depth, and transfer row-group size from latency targets and measured link capacity.

### Evidence

On Qwen3-30B-A3B, shrinking token chunks from 4,096 to 512 increased stage compute time by 133.6%; shrinking to 128 increased it by 591.7%. This is the key measurement behind the design: narrow chunks are not merely smaller GEMMs—they can force repeated MoE expert-weight traffic.

The fused codec combines residual and hidden states, applies a Walsh–Hadamard rotation, packs to INT4, and reportedly:

- takes about 0.10 ms for small batches;
- reduces transfer payload by 7.53×;
- changes MMLU by +0.06 percentage points and GSM8K by −0.46 points versus BF16 transport.

Across A800, RTX 4090, and RTX 3090 configurations, the paper reports:

- 4.44× the SLO goodput of a fixed 4,096-token baseline in a four-stage MoE pipeline;
- 23.3× the SLO goodput of calibrated Revisit-PP in that setup;
- p95 TTFT of 509 ms versus 53.9 seconds for Revisit-PP on its 32K MoE workload;
- all 74 requests meeting their latency targets during a synthetic tenfold bandwidth swing.

### Why this matters for CPU and EPYC deployments

The transferable idea is broader than GPU pipeline parallelism: **compute blocking, interruption granularity, and communication framing should be separate control variables.**

For a CPU or hybrid EPYC deployment, analogous coupling appears when one tile or token chunk simultaneously determines:

- GEMM efficiency and weight-cache reuse;
- scheduler preemption points;
- NUMA transfer size;
- CPU–GPU activation movement;
- RPC or RDMA message size.

A CPU-oriented prototype should retain a large logical prefill batch but execute it in layer or operator slices, with independently framed activation transfers. The scheduler should choose among pre-measured operating points using:

- per-layer kernel time by shape;
- local and remote DRAM bandwidth;
- link bandwidth and queue depth;
- weight residency or LLC reuse;
- decode latency slack.

The important comparison is not only against a fixed token chunk. Compare with the best independently tuned token chunk for each workload and measure p99 TTFT/ITL, aggregate goodput, weight-read bandwidth, and scheduler overhead.

### Caveats

- The implementation is built on vLLM and Triton; it does not evaluate CPUs.
- Ethernet experiments shape links to 100–1,000 Mbps with up to 30 ms RTT. Results may change on high-speed RDMA fabrics where codec overhead is less attractive.
- The largest multipliers are SLO-goodput ratios against baselines that badly miss latency targets; they are not raw throughput multipliers.
- Only 16 of 21 requests met targets in the cited four-stage 32K MoE replay, despite the large relative improvement.
- INT4 activation transport is lossy. The reported quality changes are small, but broader model and workload validation is necessary.
- No maintained public implementation was identified with the paper.

**Verdict:** a substantive scheduling contribution. Its strongest idea is the three-way decoupling, not the headline 23.3× figure.

## 2. DFly2: reuse the target’s cache and its sparse index

**Importance: High for long-context speculative decoding**

[DFly2: Turbocharging Long-Context Speculative Decoding by Sharing the Target’s Sparsity](https://arxiv.org/abs/2611.34776), submitted November 25, addresses a growing inefficiency in speculative decoding for natively sparse models.

A conventional draft model often maintains its own KV state. A shared-KV drafter can remove that duplicate cache, but it still needs to decide which positions to read. Dense reads scale with context; a new sparse-selection pass adds cost and state.

DFly2 reuses two artifacts already paid for by the target:

1. **KV sharing:** each draft layer reads the cache produced by a paired target layer.
2. **Index sharing:** it reads only positions selected by that target layer’s sparse-attention indexer.

The drafter therefore keeps no private KV cache and runs no independent indexer. The paper’s key empirical premise is that a sparse selection computed one step earlier still covers close to 90% of the positions selected by later draft tokens on DeepSeek-V4.1-Flash.

### Reported results

The method is evaluated on DeepSeek-V4.1-Flash, GLM-5.3-Flash, and Hy4 flash, which use different sparse-attention designs.

Relative to drafts trained with the same recipe:

- accepted-length advantage over the stronger baseline grows from 3.9–10.5% on general workloads to 10.0–31.5% on long-context and agentic workloads;
- on two 64K DeepSeek-V4.1-Flash workloads, high-concurrency throughput at matched per-user decode speed improves by 62% and 81%;
- for DeepSeek-V4.1-Flash, a three-layer drafter’s own cache would add 3.4 KiB/token to a 4.4 KiB/token target cache—a 1.76× total KV footprint;
- on long-context tests, DFly2 reports group-average end-to-end speedup of 2.11× over autoregressive decoding, versus 1.91× for DSpark and 1.52× for DFlash.

The distinction between decode and end-to-end speed remains important. On agentic tasks, one DFlash configuration can decode much faster while producing less than 1× end-to-end speedup because its prefill and private-cache costs dominate. DFly2 specifically attacks those hidden costs.

### Practical implications for CPU inference

For a memory-bandwidth-bound CPU decoder, duplicate draft state is especially expensive. Every private draft KV page competes with target weights and KV for DRAM bandwidth and capacity. DFly2 suggests a general rule:

> An auxiliary inference path should consume the target’s already-materialized state and routing decisions whenever their staleness is tolerable.

A CPU implementation should test:

- target-only decode;
- a drafter with private KV;
- shared target KV with dense or windowed reads;
- shared KV plus stale target indices.

Measure total resident bytes, bytes read per accepted token, target passes per output token, cache locality, and concurrency at fixed p99 ITL. On multisocket EPYC, the shared index should also encode placement: a slightly lower-value local KV block may be preferable to a remote-socket block once access cost is included.

The staleness result is also useful beyond speculative decoding. Sparse-attention indices may be reusable across several decode steps, draft positions, or related branches, provided the runtime monitors coverage degradation.

### Availability and caveats

[AngelSpec](https://github.com/Tencent/AngelSpec) provides the training framework, and released DFly2 checkpoints include [DeepSeek-V4.1-Flash-DFly2-B6](https://huggingface.co/AngelSlim/DS-V4.1-Flash-DFly2-B6). However, the checkpoint documentation says vLLM serving support is still in progress.

Other limitations:

- DFly2 depends on a target with an exposed sparse indexer and compatible per-layer KV.
- Results are for GPU deployments and model-specific paired layers.
- Index staleness may behave differently under sampling, distribution shifts, or longer draft blocks.
- The target index is not free in an absolute sense; DFly2 avoids duplicating a cost the target already incurs.
- Throughput comparisons need matched user latency and memory limits, because the main benefit is increased concurrency rather than only faster batch-one decode.

**Verdict:** a genuine architectural improvement for sparse long-context models. It is more compelling than another larger draft because it removes redundant state and computation from the serving graph.

## 3. Suffix KV reuse enables efficient editable contexts

**Importance: Medium-high for agent serving**

[Letting Models Manage Their Own Context: Where We Stand](https://arxiv.org/abs/2611.34147), submitted November 25, evaluates agents that decide when and what to compact through a context API rather than relying on a harness to summarize automatically.

The inference-systems contribution is narrower but important. Standard prefix caching stops helping at the first edited token. If an agent replaces an old span with a summary, unchanged events after the edit are normally prefetched again because their positions and preceding context changed.

The authors prototype **suffix KV reuse** in SGLang: cached state for unchanged post-edit content is retained and adjusted rather than completely recomputed. With Qwen3.6-27B, this reduces per-request prefill compute by 4–15% across OfficeQA Pro, MLE-bench Lite, and DeepSWE, with no measurable accuracy change.

### Why it matters

Agent contexts are becoming mutable data structures rather than append-only strings. They contain tool results, summaries, archived events, and replacements. A serving engine that exposes only prefix-cache semantics will repeatedly recompute large unchanged suffixes.

For production design, context mutation should become an explicit API with:

- stable segment or event identities;
- a versioned edit log;
- dependency tracking for position encoding and attention state;
- validation that reused suffix state is mathematically compatible;
- fallback recomputation when edits cross unsupported model-state boundaries.

For CPU inference, avoiding 4–15% of prefill work can be meaningful because long agent prompts are compute-heavy and can disrupt decode latency. The opportunity grows if the unchanged suffix is large and reused state remains local to the same NUMA node.

### Caveats

- This is a prototype, not a standardized serving interface.
- The savings depend on edit location and context structure.
- Correct reuse is model-dependent: RoPE positions, recurrent layers, sliding-window attention, and hybrid state-space layers complicate suffix validity.
- The result reports reduced prefill work, not a full cluster-level cost or tail-latency study.
- The paper’s large agent-quality gains come from model-driven context management, not solely from suffix KV reuse.

**Verdict:** not as mature as PipeAxes or DFly2, but strategically important. Editable-context semantics should be considered now when designing KV-cache APIs for agent serving.

## Recommended reading order

1. [PipeAxes](https://arxiv.org/abs/2611.34615) — focus on the motivation measurements, layer slicing, transport codec, and SLO-goodput methodology.
2. [DFly2](https://arxiv.org/abs/2611.34776) — read the cache-footprint analysis, index-staleness measurements, and matched-speed throughput results.
3. [AngelSpec repository](https://github.com/Tencent/AngelSpec) and [DFly2 checkpoint](https://huggingface.co/AngelSlim/DS-V4.1-Flash-DFly2-B6) — useful for determining what is reproducible today.
4. [Letting Models Manage Their Own Context](https://arxiv.org/abs/2611.34147) — read the context API and suffix-reuse section as an inference-engine design proposal.

## Bottom line

The new common theme is **reuse decisions, not just tensors**.

- PipeAxes preserves the compute-efficient batch while reusing its work across finer scheduling slices.
- DFly2 reuses both target KV and the target’s sparse-selection decision.
- Suffix KV reuse preserves state for unchanged context even after an earlier span is edited.

For an EPYC-oriented system, the most valuable prototype is a scheduler that treats compute tiles, preemption slices, and NUMA/network transfer frames as independent variables. The second is a shared-state speculative path whose drafter has no private long-context cache. Both should be evaluated at fixed p99 user latency and memory capacity, because their real benefit is higher SLO-constrained concurrency—not merely a better isolated kernel time.