# AI inference systems briefing — November 23, 2026

## Executive assessment

The most important new work since Friday changes how inference platforms should be validated and secured.

[Validation-driven Deployment of LLM Inference Platforms for Agentic Workloads](https://arxiv.org/abs/2611.25905), from IBM Research and Red Hat AI, finds that public traces substantially overestimate useful capacity for real agent workloads. In a four-model MoE rollout on llm-d, captured workloads reached about half the “optimal” concurrency and roughly 60% of the throughput suggested by public traces. The paper’s stronger methodological result is that captured-session replay gives repeatable load while live agent reruns do not, and that live, replayed, and synthetic tests exercise complementary serving paths.

[BlockJack](https://arxiv.org/abs/2611.25945) demonstrates that a paged-KV block table is a security-critical address-translation structure. A physical fault that changes a mapping can redirect one tenant’s attention reads into another tenant’s KV blocks, causing both information leakage and output manipulation. This matters directly to any runtime that has adopted page-table-like cache management.

For an engineer accustomed to CPU performance regression testing, the shared lesson is familiar: a serving platform cannot be qualified by peak tokens/s alone. It needs representative replay, path coverage, fault containment, and validation of the metadata that binds logical requests to physical state.

## 1. Captured agent traces expose false capacity confidence

**Importance: High for platform sizing, regression testing, and production rollout**

[Validation-driven Deployment of LLM Inference Platforms for Agentic Workloads](https://arxiv.org/abs/2611.25905), submitted November 19, proposes a lifecycle-wide validation pipeline spanning small-scale tests, sizing, pre-production, and production.

The problem is that agentic workloads are difficult to reproduce. A live rerun of the same task can take different branches, issue different tool calls, and generate radically different token counts. A synthetic request generator may reproduce a length histogram while missing session correlation, cache reuse, tool pauses, multimodal preprocessing, or rare control paths. Public traces may be internally consistent yet unlike the workload being deployed.

The paper separates three test modes:

| Test mode | Best use | Main limitation |
|---|---|---|
| Live agent rerun | End-to-end behavior and quality | Nondeterministic load and expensive execution |
| Captured-session replay | Repeatable performance and capacity testing | Cannot discover new agent trajectories |
| Synthetic/public workload | Broad stress and parameter sweeps | May misrepresent production correlations and code paths |

The modes are complementary, not interchangeable.

### The most consequential measurements

At the 95th percentile, captured replay stays within **1.18×** of the original session’s recorded input-token load. Live reruns reach **4.98×**, showing that repeated execution of the same agent task is not a stable performance benchmark.

In a case study deploying four large MoE models on llm-d:

- captured workloads reached about **half the concurrency** that looked optimal under public traces;
- realized throughput was about **60% of the public-trace result**.

That is not a small benchmarking error. A platform sized from the public workload could miss its expected capacity by roughly 40%, or be configured at a concurrency that raises tail latency and destabilizes the real application.

The authors also measure Python code coverage in vLLM and find that live, replayed, and synthetic tests each reach serving paths not covered by the others. This is important evidence against the common practice of using one “representative” benchmark as both a performance test and a release qualification suite.

### Why it matters for CPU and EPYC inference

CPU inference is especially sensitive to workload correlations that summary statistics erase:

- prompt length affects prefill shape and packing efficiency;
- output length affects decode residency and KV growth;
- simultaneous long contexts determine memory-bandwidth saturation;
- prefix locality changes both compute demand and NUMA placement;
- model or adapter mix changes weight residency;
- tool pauses change batch composition and cache lifetime;
- request concurrency alters frequency, thread contention, and LLC behavior.

A synthetic generator that independently samples these fields may match every marginal histogram and still produce the wrong system behavior.

For a ZenDNN or EPYC serving stack, captured replay should preserve at least:

```text
session_id
arrival and pause timing
model / adapter / prompt-template identity
input and output token counts
prefix lineage or cache key
tool-interaction boundaries
cancellation and retry events
multimodal preprocessing metadata
priority and latency objective
```

Payloads can be redacted or replaced, but the causal structure should remain intact. If token content changes, verify that tokenization, sparsity, routing, and prefix sharing still resemble production.

### A stronger validation ladder

A practical release pipeline should use four layers:

1. **Kernel and operator tests**  
   Differential correctness, numerical tolerance, ISA coverage, and shape-level performance.

2. **Captured trace replay**  
   Repeatable capacity, TTFT, TPOT, memory, and cache-behavior comparisons.

3. **Synthetic adversarial stress**  
   Extreme context lengths, burst arrival, model-mix shifts, cancellation storms, NUMA imbalance, and memory-tier failures.

4. **Small live canary**  
   Real agent branching, quality, tool interactions, and unanticipated runtime paths.

Do not promote a change merely because it wins a synthetic throughput benchmark. Require replay parity and then verify that the live canary’s token, cache, and latency distributions remain inside an expected envelope.

### Metrics that matter

In addition to throughput and p99 latency, collect:

- per-session tokens and wall time;
- useful concurrency before SLO violation;
- prefix-cache hit rate and reused bytes;
- local versus remote NUMA reads;
- DRAM bandwidth and effective CPU frequency;
- allocator and cache occupancy by state class;
- request cancellation and retry rate;
- code-path or feature coverage;
- deviation between captured and replayed workload statistics.

For regression triage, store the complete configuration fingerprint: model revision, quantization, tokenizer, runtime commit, kernel library, CPU microcode, BIOS NUMA mode, thread affinity, huge-page policy, and memory-tier settings.

### Assessment and limitations

This is a substantive engineering paper because it validates the validation process itself and releases tools and a trace dataset. The finding that realistic captured workloads achieve only 60% of public-trace throughput is more operationally valuable than another isolated kernel speedup.

The case study still covers four MoE models on llm-d, so the exact ratios should not be generalized to dense CPU deployments. Code coverage also indicates that a path executed, not that its states or failure modes were thoroughly tested. The transferable result is the multi-mode validation methodology.

**Read:** [Validation-driven Deployment of LLM Inference Platforms for Agentic Workloads](https://arxiv.org/abs/2611.25905)

## 2. BlockJack: paged-KV metadata is a tenant-isolation boundary

**Importance: High for shared inference services; indirect but important for performance-runtime design**

[BlockJack: Cross-Tenant Redirection Fault Attacks on Paged KV Cache Address Translation in LLM Serving](https://arxiv.org/abs/2611.25945), submitted November 19, targets the block tables used by paged KV-cache systems such as vLLM.

These tables map a request’s logical KV blocks to physical cache blocks. That makes them analogous to operating-system page tables, but they are maintained by the serving runtime and optimized for rapid scheduling and kernel consumption.

BlockJack corrupts a block-table entry so that one request reads KV state assigned to another tenant. It does not need to modify the model, prompt, or cached tensor data.

The paper demonstrates two consequences:

- **Confidentiality violation:** victim context appears in an attacker-controlled response.
- **Integrity violation:** attacker-provided context influences a victim’s output.

Across seven models from the Qwen, Llama, Gemma, and Phi families, redirected mappings expose victim context. In a 248-scenario medical benchmark, the integrity attack causes up to **41.1%** of responses to recommend less urgent care while remaining superficially consistent with the victim’s question.

The authors demonstrate physical feasibility using Rowhammer and evaluate assignment validation as a defense. vLLM and Sonar confirmed the vulnerability through responsible disclosure and proposed patches reviewed by the authors.

### Why this matters beyond Rowhammer

The narrow attack uses a physical memory fault, but the engineering implication is broader. Any bug or corruption that changes a request-to-block mapping can become cross-tenant state confusion:

- use-after-free or stale block-table entries;
- allocator races;
- incorrect prefix-cache deduplication;
- failed migration between cache tiers;
- checkpoint or restore bugs;
- DMA writing the wrong descriptor or page;
- silent memory faults;
- missing cleanup after cancellation.

Paged KV metadata should therefore receive protections comparable to memory-management metadata, not ordinary performance bookkeeping.

### Practical hardening for CPU and hybrid runtimes

A high-performance validation scheme should avoid scanning every mapping on the critical path. Useful options include:

1. **Ownership tags**  
   Store tenant/request generation IDs with physical blocks and validate them when constructing a batch.

2. **Generation counters**  
   Increment block generations on reuse so stale mappings fail even when a physical block number is recycled.

3. **Range and cardinality checks**  
   Verify block IDs, expected sequence length, and mapping count before kernel launch.

4. **Protected metadata placement**  
   Separate block tables from large, frequently rewritten KV payloads; use ECC-capable memory where available.

5. **Integrity summaries**  
   Maintain checksums or MACs over mapping pages at trust boundaries such as tier restore, migration, or cross-process handoff.

6. **Zeroization and quarantine**  
   Clear released blocks before cross-tenant reassignment, and quarantine blocks after detected corruption.

7. **Fault-injection tests**  
   Mutate mappings, generations, ownership tags, and transfer descriptors to verify fail-closed behavior.

For CPU inference, metadata validation can often be performed by the scheduler thread before dispatching work to compute teams. Batch the checks and keep ownership metadata cache-resident. Measure the overhead rather than assuming security necessarily conflicts with throughput.

### Performance implications

Some optimizations increase the attack and failure surface:

- cross-request prefix sharing;
- parallelism-independent secondary cache identities;
- remote or object-store KV tiers;
- zero-copy page remapping;
- asynchronous migration;
- compact block-table formats;
- shared CPU cache regions across replicas.

Each mechanism introduces another point where logical identity must remain bound to physical state. The right abstraction is a typed cache handle containing model, tenant, request lineage, block generation, layout, and precision—not a bare physical block index.

### Assessment and limitations

BlockJack is a genuinely important deployment result. It demonstrates an end-to-end confidentiality and integrity failure, physical feasibility, multi-model impact, and a concrete defense.

Its immediate exploitability depends on the attacker’s ability to induce a physical fault and predict useful mapping targets. That is a stronger assumption than ordinary remote access. Still, the paper exposes a real architectural weakness: paged-KV translation lacks the mature protection mechanisms applied to hardware page tables.

**Read:** [BlockJack: Cross-Tenant Redirection Fault Attacks on Paged KV Cache Address Translation in LLM Serving](https://arxiv.org/abs/2611.25945)

## 3. Secondary signal: low-bit prefill reaches native INT4 execution

[BitPrefill](https://arxiv.org/abs/2611.26657) proposes a W1.58A4 operator for BitNet-b1.58 that maps ternary weights and 4-bit activations onto native INT4 tensor-core GEMM. On an RTX 4090 and Jetson Orin Nano, the authors report up to **1.42× faster prefill** than BF16 with negligible reported accuracy loss.

The useful observation is that quantization paths optimized for memory-bound decode may leave compute-bound prefill at BF16. Prefill and decode require separate kernel and precision strategies.

For CPU work, the analogous question is whether a low-bit model actually uses AMX-INT8 or another native dot-product path during large-token prefill, or pays enough unpacking, scaling, and conversion overhead to erase the theoretical gain. A credible CPU port would need end-to-end TTFT, packing amortization, and accuracy measurements across prompt lengths—not only GEMM timing.

This is technically interesting but not yet a priority result for EPYC inference: it targets one ternary model family and GPU/edge tensor cores, and the 1.42× gain is much smaller than the nominal precision reduction suggests.

**Read:** [BitPrefill: W1.58A4 Tensor-Core Acceleration for Compute-Bound LLM Prefill](https://arxiv.org/abs/2611.26657)

## Recommended engineering actions

1. **Add captured-session replay to performance qualification.** Preserve timing, lineage, cache reuse, pauses, and request mix—not only token-length histograms.
2. **Use live, replayed, and synthetic tests together.** Track which serving features and failure paths each mode exercises.
3. **Size from realistic goodput.** Select concurrency using captured workloads and latency objectives rather than peak throughput from public traces.
4. **Promote KV mappings to protected metadata.** Add ownership, generation, layout, and tenant identity to cache handles.
5. **Build block-table fault injection.** Corrupt mappings, restore metadata, and reuse generations; verify that requests fail closed without exposing another tenant’s state.
6. **Benchmark security overhead explicitly.** Batch assignment validation and measure scheduler CPU cost, LLC behavior, and end-to-end TPOT.
7. **Optimize prefill and decode independently.** Confirm that low-bit formats reach native compute instructions in both phases.

## Bottom line

Friday’s briefing focused on moving cache state and communication efficiently. The weekend’s strongest work adds two constraints: the workload used to validate the system must resemble production, and the metadata that locates state must preserve tenant identity under faults.

For an EPYC inference platform, the highest-value immediate work is a replay-driven qualification harness tied to a hardened cache-handle design. Use captured agent sessions to establish realistic concurrency and tail latency, then inject mapping and ownership faults into the same runs. That combination tests both the performance contract and the isolation contract of the serving runtime—two properties that peak tokens/s cannot establish.