## Inference systems update — October 14, 2026

### vLLM makes prefill/decode KV transfer bidirectional for multi-turn agents

**Importance: High for disaggregated, long-context serving; indirect but actionable for CPU and heterogeneous systems**

One substantive implementation-level development warrants adding to the previous briefing. On October 12, vLLM published a reproducible design for returning KV state from decode nodes to prefill nodes between conversation turns.

Conventional prefill/decode disaggregation transfers KV only from prefill to decode. After generation, the decode worker owns KV for both the original prompt and newly generated tokens. On the next turn, a prefill worker that lacks those blocks must recompute part—or all—of the accumulated history. The new path adds a reverse decode-to-prefill read, coordinated by a KV-aware router. genui{"citation":{"ref":"turn46view0"}}

The control flow is:

1. The decode worker retains completed-request KV blocks for a configurable TTL.
2. It returns block-location metadata in the terminal response.
3. The router indexes that metadata by a client-provided conversation ID.
4. On the next turn, prefill first checks its local prefix cache.
5. If the remaining remotely reusable prefix exceeds a threshold, prefill issues an asynchronous NIXL read from decode.
6. Prefill computes only the uncached suffix; the router never carries the KV tensors themselves.

Two controls are especially important:

- `decoder_kv_blocks_ttl` defaults to 480 seconds and trades decode-side cache capacity for cross-turn reuse.
- `kv_recompute_threshold` defaults to 64 tokens and avoids transferring a short prefix when local recomputation is cheaper.

The threshold is the right abstraction, but a fixed token count is not portable. The decision boundary depends on model shape, KV format, interconnect topology, current congestion and whether the source memory is local to the NIC or accelerator.

A more robust policy is:

[
	ext{reuse remotely if } T_{	ext{lookup}} + T_{	ext{transfer}} + T_{	ext{install}}
<
T_{	ext{recompute}}
]

Both sides should be estimated online. A 64-token cutoff that is sensible over EFA between H200 nodes may be wrong for CPU DRAM, CXL memory, PCIe-attached accelerators or a congested fabric.

### Reported performance

The published benchmark used a 1-prefill/1-decode deployment on two EC2 `p5en.48xlarge` instances, each with eight H200 GPUs and 16 EFA interfaces. With prefill-side prefix caching disabled to emulate eviction or routing to a cold prefill node:

- Qwen3-32B TTFT rose from 122 ms at 2K input tokens to 460 ms at 24K with bidirectional transfer, versus 158–778 ms with recomputation.
- GLM-5-FP8 TTFT rose from 358 ms at 2K to 702 ms at 20K, versus 500–2,334 ms without reverse reuse.
- The largest reported TTFT reductions were 1.6× for Qwen3-32B and 3.3× for GLM-5-FP8.
- Reverse-transfer time was roughly 30–130 ms for Qwen3-32B and 29–110 ms for GLM-5-FP8, remaining below the prefill compute it replaced. genui{"citation":{"ref":"turn46view0"}}

**Read:** [vLLM’s bidirectional KV-transfer writeup and reproduction guide](https://vllm.ai/blog/2026-10-12-bidirectional-kvxfer-multiturn-agentic-workload)

### Why it matters in practice

This turns disaggregated serving from a request-level pipeline into a stateful multi-turn system. The scheduling objective can no longer be only “send this request to the least-loaded prefill and decode pair.” It must also price:

- where the conversation’s newest KV blocks reside;
- whether the retained blocks will survive until the next turn;
- the cost of moving them versus recomputing them;
- the decode-memory opportunity cost of retention;
- fabric and NUMA locality along the transfer path.

For CPU and EPYC deployments, the same mechanism applies even if one endpoint is host DRAM rather than another GPU. A decode worker could retain hot conversational state in socket-local DRAM, while a later prefill runs on another CPU socket or accelerator. The placement policy should distinguish:

[
	ext{local DRAM} ; / ; 	ext{remote NUMA DRAM} ; / ; 	ext{CXL} ; / ; 	ext{RDMA peer}
]

A cache hit is not uniformly valuable across those tiers. The router should rank candidate workers by expected service time, not hit count.

The most useful EPYC experiment would compare four paths for realistic multi-turn traces:

1. local prefix-cache reuse;
2. remote-socket KV reuse;
3. cross-node RDMA reuse;
4. local recomputation.

Record TTFT by turn number, bytes transferred per reused token, local and remote memory bandwidth, NIC locality, retained-KV occupancy, eviction rate and decode throughput lost to retention. Then fit an online reuse/recompute boundary per model and topology.

### Caveats

The result is substantive, but the headline speedups are not general-purpose P/D gains:

- Prefill prefix caching was deliberately disabled, so the experiment represents cold or evicted prefill workers rather than an affinity-perfect deployment.
- The topology is a high-bandwidth H200/EFA configuration; transfer economics will differ sharply on ordinary Ethernet, PCIe-only systems and CPU memory tiers.
- A fixed TTL can strand valuable decode capacity when conversations go idle and expire useful state just before a delayed next turn. Lease renewal or reuse-probability-based eviction would be stronger.
- Conversation IDs become correctness and isolation boundaries. Multi-tenant deployments must bind metadata to tenant, model revision, tokenizer, position encoding, KV dtype and layout—not trust a client identifier alone.
- The article says the capability is available in vLLM 0.21.0 or later, while its appendix installs v0.29.0 and additionally applies PR #40186 so streaming responses return transfer metadata. Reproduce from the exact documented commit set rather than assuming an arbitrary released image contains the complete path. genui{"citation":{"ref":"turn46view0"}}

### Assessment

This is a genuinely useful systems advance, not merely a benchmark claim. It eliminates a structural inefficiency in multi-turn P/D serving and exposes the right policy question: when should state move, remain resident or be recomputed?

For an engineer working on CPU inference and large-scale serving, the valuable next step is to combine this mechanism with the earlier KV working-set analysis: predict whether a conversation will return, retain its state in the cheapest tier that still beats recomputation, and route the next turn using topology-aware transfer cost. That is more promising than treating prefix-cache capacity, request routing and prefill/decode placement as separate optimizations.