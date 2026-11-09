# AI inference systems briefing — November 9, 2026

## Executive take

The most important development since the November 6 StrataFlux briefing is **FlashOffloading**, a host-DRAM-centric KV-cache architecture that fully offloads live KV state during both prefill and decode while using a different device-residency policy for each phase.

[FlashOffloading: Low-Latency KV Cache Offloading for Long-Context LLM Serving](https://arxiv.org/abs/2611.07542), submitted November 5, combines:

- four rolling full-layer buffers for prefill;
- a token-level hot cache for sparse decode;
- a reverse token-to-slot index whose management cost does not grow with hot-cache capacity;
- fused index/cache-management and transfer/attention kernels;
- one host-backed KV history shared across a prefill/decode-disaggregated deployment.

On a 64-die Ascend 910C cluster serving GLM-5.2 with MTP3 speculative decoding, the paper reports up to 1.38× higher prefill throughput and 1.91× higher decode throughput under a 30 ms TPOT constraint. It also makes 512K- and 1M-token requests servable in configurations where device-resident KV cannot fit. The implementation is integrated into [vLLM-Ascend 0.30](https://github.com/vllm-project/vllm-ascend/tree/releases/v0.30.0).

The important result is not “move KV to CPU memory.” That is established practice. The contribution is a demonstration that **offloading must be phase-specialized all the way down to buffer lifetime, index direction, transfer granularity, and kernel structure**. Prefill has coarse layerwise transfers and long compute windows; decode has fine-grained, data-dependent reads that become known only after sparse indexing. A single cache policy cannot optimize both.

For CPU inference engineering, the strongest lesson is that a larger DRAM cache can be operationally cheap only if lookup and eviction cost depend on the active miss set—not on total cache capacity. This directly applies to NUMA-local KV working sets, CXL-backed caches, and CPU–accelerator serving.

**Assessment:** a substantive end-to-end result with open-source integration, useful ablations, and explicit SLO measurements. Its decode path depends on native sparse attention and specialized Ascend kernels, so the reported gains do not transfer directly to dense models, GPUs, or CPU-only inference.

## 1. What changed since StrataFlux

Friday’s StrataFlux result used a complete low-bit resident model and KV cache, with slower tiers supplying optional precision overlays. A missed refinement could be skipped without blocking correctness.

FlashOffloading solves a different problem:

| System | Fast-tier guarantee | Slow-tier role | Miss behavior |
|---|---|---|---|
| StrataFlux | Complete low-bit weights and KV remain usable | Supplies optional precision refinements | Skip late refinement and continue at base precision |
| FlatCake | KV scope is assigned to DRAM or SSD at creation | Permanent home for selected scopes | Retrieve or recompute according to placement |
| FlashOffloading | Only the phase-specific active KV working set remains on device | Host DRAM holds the complete exact KV history | Required sparse-attention misses must be fetched before use |

FlashOffloading therefore has a harder latency contract. It preserves exact model-defined sparse attention: every token selected by the native indexer must be available to attention. It cannot simply omit a late miss. The system must reduce miss count, make lookup cheap, and overlap required transfers.

This makes the work complementary to StrataFlux rather than a replacement. A future system could keep exact low-bit KV in host memory while using optional precision overlays in colder tiers, but the scheduling and quality contracts would need to remain distinct.

## 2. Prefill: exploit layerwise lifetime with four device buffers

During chunked prefill, every layer owns a growing KV history, but only the current layer consumes its full history at a given moment. FlashOffloading stores committed KV in host DRAM and allocates only four full-layer device buffers:

- **B0** remains permanently bound to layer 0.
- **B1–B3** rotate across the remaining layers.
- H2D loading, layer computation, and D2H writeback use separate streams when the hardware supports them.
- A binding table and device events prevent a buffer from being reused until both its compute consumer and writeback have completed.

Keeping layer 0 resident is a deliberate pipeline bootstrap. Compute begins immediately on B0 while layer 1 loads, creating the first overlap window. For later layers, the preload target need not be (i+1); the runtime can start several layers ahead when transfer time exceeds one layer’s compute window.

For a model with (L) KV-owning layers, sequence length (T), and per-token per-layer KV size (s_{mathrm{KV}}):

[
M_{mathrm{resident}} = LTs_{mathrm{KV}}
]

whereas the dominant FlashOffloading prefill allocation is:

[
M_{mathrm{prefill}} approx 4Ts_{mathrm{KV}}
]

For GLM-5.2 with 78 KV layers, the dominant KV fraction is approximately:

[
rac{4}{78} approx 5.1%
]

The exact total is higher because indexer state, block tables, metadata, and activations remain, but model depth no longer multiplies the dominant live KV allocation.

### Why reduced KV can lower TTFT

At the same chunk size, offloading is slower because reused prefix state is reloaded during each forward pass. The paper reports this negative result directly. Offloading wins when the freed device memory permits a much larger chunk:

- larger chunks improve accelerator utilization;
- fewer chunks reduce scheduling and repeated model-forward overhead;
- sufficiently long requests become feasible at all.

In the 256K-token prefill ablation, FlashOffloading reduces KV HBM usage from 36.75 GB to 9.04 GB and increases the supported batched-token limit from 16,384 to 131,072. The practical optimization target is therefore not transfer latency in isolation:

[
T_{mathrm{TTFT}}
=
N_{mathrm{chunks}}
left(
T_{mathrm{forward}}
+
T_{mathrm{transfer,exposed}}
+
T_{mathrm{scheduler}}
ight)
]

A modest transfer cost can win if the capacity saving reduces (N_{mathrm{chunks}}) enough.

## 3. Decode: reverse the index so capacity no longer raises lookup cost

GLM-5.2’s native sparse indexer selects 2,048 historical tokens per attention step. The complete KV history remains in host DRAM, while HBM holds a hot subset.

The obvious representation maps each hot-cache slot to its token. Testing whether the selected top-(k) tokens are resident then requires a hash/intersection or a scan over cache entries. Prior schemes consequently face a bad tradeoff:

- a small hot cache causes more H2D misses;
- a large hot cache lowers misses but makes lookup and eviction more expensive.

FlashOffloading reverses the mapping. For every historical token, **cache_slots[token]** records its HBM slot or (-1) if absent. Residency lookup becomes (O(1)) per selected token.

The management complexity changes from approximately:

[
O(k) + O(|C|)
]

to:

[
O(k) + O(|mathrm{Miss}|)
]

where (C) is hot-cache capacity and (|mathrm{Miss}| ll |C|).

Eviction and index updates also touch only the miss set. This decouples cache-management latency from capacity, allowing the hot-cache ratio to grow from roughly one-eighth to one-quarter of the history and supporting nearly four times larger batches in the authors’ configuration.

This is the most directly transferable idea for CPU inference.

### CPU and NUMA implication

Suppose a multisocket CPU server keeps a request’s complete KV history in remote DRAM or CXL memory and a hot working set in socket-local DRAM. A slot-to-token directory forces local-cache lookup work to scale with the local cache’s size. A token-to-location directory makes every selected token answer:

[
	ext{location}(t)
in
{
	ext{local slot},
	ext{remote socket},
	ext{CXL},
	ext{storage}
}
]

in constant time.

The directory can carry more than a slot:

- NUMA node;
- page or block identifier;
- tier;
- precision;
- generation/version;
- in-flight transfer state.

The cost is metadata proportional to total context length. FlashOffloading acknowledges this: KV payload shrinks substantially, but the sparse index, reverse directory, and other metadata still grow with (T). On CPU, keep the hottest directory region local, use compact entries, and measure TLB and cache pressure. A theoretically constant lookup that misses LLC on every selected token can still dominate.

## 4. Fuse residency lookup into top-k selection

Even with (O(1)) directory access, running a separate miss-lookup kernel after top-k adds another pass over selected indices and another synchronization point.

FlashOffloading’s LightningIndexerManage kernel encodes each candidate token’s index together with its cache slot before top-k. The pair travels through selection with the score. The winning entries therefore emerge with both:

- the token index needed for exact attention;
- the current HBM slot, or (-1) indicating a miss.

Miss detection adds no separate post-top-k lookup pass. The kernel also produces copy descriptors for the following transfer/attention stage.

On the evaluated operating point, the extra cache-management latency is reported at 106–232 microseconds for an indexer layer. Only 21 of GLM-5.2’s 78 layers run the indexer, yielding roughly 3.4 ms per decode step against an approximately 100 ms total inter-token latency.

For CPU kernels, the same fusion principle applies:

- attach residency metadata while building the selection candidates;
- carry it through partial top-k reductions;
- emit hit runs and miss descriptors directly;
- avoid materializing and rereading a standalone list of selected token IDs.

This is a promising AVX-512/AMX support-kernel target even when attention itself remains in a library backend. The key is to preserve sequential directory access where possible and prevent the metadata payload from degrading the top-k vectorization.

## 5. Overlap required misses inside sparse attention

After selection, missing KV must move from pinned host memory to the accelerator. A conventional pipeline performs:

1. select tokens;
2. copy misses into hot-cache slots;
3. launch sparse attention;
4. gather the selected KV from HBM into an attention workspace.

That serializes transfer and adds a redundant HBM pass for newly fetched misses.

FlashOffloading’s CopySFA kernel combines the copy and sparse-attention gather. Sparse attention divides the selected 2,048 tokens into four tiles. The implementation groups misses at the head, so only one tile performs host-DRAM movement. While that tile loads, compute proceeds on hit tiles.

The fused data path changes from:

[
	ext{DRAM read}
ightarrow
	ext{HBM slot write}
ightarrow
	ext{HBM slot read}
ightarrow
	ext{workspace write}
]

to:

[
	ext{DRAM read}
ightarrow
{
	ext{HBM slot write},
	ext{workspace write}
}
]

The paper reports 1.03–1.27× kernel-level speedups over separate copy and attention kernels, with 90–98% of copy time overlapped at batch sizes of at least 16.

### CPU analogue

A CPU-only implementation has no H2D boundary, but the same problem occurs across NUMA and CXL tiers. Do not first copy a remote KV row into a local cache and then reread it into the attention kernel. Instead:

- fetch the remote cache line or block into the attention tile’s working buffer;
- update the local hot-cache copy as a side effect;
- compute on already-local hits while remote reads progress;
- merge tile softmax statistics normally.

This requires a backend that exposes tile scheduling and asynchronous movement. A black-box attention primitive cannot easily express it. The likely implementation point is a custom sparse-attention kernel or an extended oneDNN/ZenDNN primitive with a miss-descriptor input.

## 6. End-to-end results and their real meaning

The end-to-end deployment uses:

- four servers;
- 32 Ascend 910C cards / 64 dies;
- one 32-die prefill pool and one 32-die decode pool;
- GLM-5.2 w4a8c16;
- MTP3 speculative decoding;
- input lengths from 64K to 1M;
- either one output token or 8,192 output tokens;
- a fixed 90% prefix-cache hit rate;
- burst arrivals rather than a production arrival trace.

Key outcomes include:

| Input / output | Baseline status | FlashOffloading outcome |
|---|---|---|
| 64K / 8,192 | Decode batch 4 | Batch 12; output throughput 3,787 → 6,358 tok/s |
| 128K / 8,192 | Decode batch 2 | Batch 6; output throughput 1,928 → 3,692 tok/s |
| 256K / 8,192 | Decode batch 1 | Batch 2; output throughput 994 → 1,397 tok/s |
| 512K / 8,192 | Not servable with prefill-only offload | Servable with full P+D offload |
| 1M / 8,192 | Not servable with prefill-only offload | Servable with full P+D offload |

At 64K, dual offloading triples decode batch size. Average TPOT rises from 16.9 to 30.2 ms—still near the paper’s 30 ms target—while output throughput increases substantially. This is exactly the capacity/latency exchange an SLO-aware system should expose.

The result also reveals a new bottleneck: rank imbalance. Offloading makes miss distributions differ across data-parallel ranks, so early finishers wait at MoE dispatch synchronization. The paper attributes 42.5 ms of the larger-batch step to synchronization wait. Capacity scaling has shifted the critical path from memory capacity toward load imbalance and collectives.

For CPU clusters, expect the same phase transition:

[
	ext{capacity wall}
ightarrow
	ext{remote-fetch variance}
ightarrow
	ext{barrier tail}
]

Once tiered memory admits more work, p99 performance may be governed by the rank or socket with the largest miss union, not by mean memory bandwidth.

## 7. Practical EPYC validation plan

A useful prototype should compare:

1. socket-local full KV residency;
2. remote/CXL full-history residency with no local hot cache;
3. hot cache with slot-to-token lookup;
4. hot cache with token-to-location reverse indexing;
5. reverse indexing plus fused selection;
6. fused selection plus direct remote-to-attention-tile fetching.

Sweep:

- 8K–256K contexts;
- one and two sockets;
- local DRAM, remote DRAM, and CXL if available;
- sparse top-(k) from 512 to 4,096;
- hot-cache ratios from 1/32 to 1/2;
- batch sizes 1–64;
- balanced and deliberately skewed request histories.

Measure:

- local/remote hit rate and miss-union size;
- directory footprint and LLC/TLB miss rate;
- top-k plus cache-management cycles;
- bytes fetched per selected token;
- duplicate and cache-line-amplified reads;
- exposed versus overlapped remote-read time;
- hot-cache update traffic;
- per-socket completion skew and barrier wait;
- p50/p95/p99 TPOT;
- maximum concurrency satisfying the latency SLO.

The success metric should be **SLO-compliant output throughput**, not raw batch capacity:

[
G(S)
=
max_C
left{
	ext{output tokens/s}(C)
;|;
mathrm{TPOT}_{p99}(C)le S
ight}
]

## 8. What is genuinely important versus narrow

### Genuinely important

- Prefill and decode share one host-backed history but use phase-specific residency and transfer policies.
- The paper attacks peak live prefill KV, not merely inactive or reusable prefix blocks.
- Reverse indexing makes cache-management complexity depend on selected tokens and misses rather than cache capacity.
- Residency lookup is fused into native top-k selection instead of added as a serial post-processing pass.
- Required misses are overlapped inside sparse attention without changing the model-selected token set.
- The evaluation reports TTFT, TPOT, ITL, throughput, HBM usage, batch capacity, and cases that remain unservable.
- The implementation is linked through an open-source vLLM-Ascend release.
- The paper exposes negative effects: same-chunk prefill is slower with offload, TPOT rises at higher admitted batch, and distributed imbalance becomes significant.

### Treat cautiously

- All measurements use Ascend 910C; the fused LIM and CopySFA kernels are hardware-specific.
- Decode gains depend on GLM-5.2’s native sparse indexer. Dense-attention models cannot use the same token-selective retrieval path without approximation or a different algorithm.
- The workload fixes prefix reuse at 90% and uses bursts, not a measured production arrival process.
- The longest decode experiments use one request per NPU after offloading; making a sequence servable is not the same as achieving high long-context throughput.
- Average TPOT is emphasized in the table; production adoption needs p99 TPOT and worst streaming-gap results.
- Host DRAM capacity, NUMA topology, PCIe/HCCS contention, and host-side bandwidth headroom are not characterized in enough detail for cross-platform planning.
- The paper’s 30 ms target is slightly exceeded in the 64K dual-offload result (30.2 ms), so the headline “under 30 ms” should be read as an approximate operating objective, not a strict guarantee across every row.
- Most authors are affiliated with Huawei or its Ascend organization; independent reproduction on other architectures remains important.

## Reading priority

1. **[FlashOffloading paper](https://arxiv.org/abs/2611.07542)** — Read §3–§5 for the unified architecture, four-buffer prefill path, reverse index, LIM, and CopySFA; §6.2–§6.4 contain the most useful ablations and end-to-end tradeoffs.
2. **[vLLM-Ascend 0.30 implementation](https://github.com/vllm-project/vllm-ascend/tree/releases/v0.30.0)** — Inspect the KVConnector hooks, host-block representation, and Ascend kernel integration before treating the system as reproducible.
3. **[vLLM KV Offloading guide](https://docs.vllm.ai/en/latest/features/kv_offloading_usage/)** — Useful comparison with the general CUDA/ROCm/XPU offloading connector and its CPU-primary tier.
4. **[StrataFlux](https://arxiv.org/abs/2611.08363)** — Revisit for the contrasting contract: optional precision overlays can be skipped, while FlashOffloading must deliver every model-selected KV miss.
5. **[FlatCake](https://arxiv.org/abs/2610.38355)** — Related storage-tier lesson: physical layout and delivery deadlines matter more than nominal cache presence.

## Bottom line

Friday’s briefing showed that precision can be an asynchronous, skippable overlay. FlashOffloading shows how to handle state that is **not** skippable: specialize by phase, make residency lookup independent of cache capacity, fuse metadata work into selection, and fetch misses directly into the computation that consumes them.

For CPU and EPYC inference, the highest-value idea is the reverse token-to-location directory combined with a local hot working set. It can make larger socket-local caches cheap to manage while the complete history resides in remote DRAM or CXL memory. The next challenge is no longer capacity alone. Once more requests fit, remote-fetch variance and synchronization tails become the limiting factors.

The practical target is therefore not “maximum offload” or “maximum cache hit rate.” It is the largest tiered-memory working set whose required misses can be delivered, consumed, and synchronized within the p99 token deadline.
