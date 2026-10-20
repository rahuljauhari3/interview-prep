# AI inference systems update — October 20, 2026

## Prefill/decode interleaving can trade queue starvation for execution starvation

**Importance: High for mixed-workload serving and capacity tuning; an operational finding, not yet an upstream-validated result**

A detailed vLLM report opened October 20 adds an important qualification to recent work on chunked prefill and prefill/decode interleaving. Admitting a short request quickly does not guarantee interactive service: the request can receive a good time to first token while its decode stream is starved by competing long prefills.

The report compares two complete serving stacks on two DGX Spark GB10 systems connected over 200 Gb RoCE. Both serve DeepSeek-V4-Flash-0731 with tensor parallelism across the two nodes. The test injects one to four approximately 98,620-token prefills, then adds short 11-token prompts three seconds later. Randomized long-prompt prefixes prevent prefix caching from masking prefill cost. genui{"citation":{"ref":"turn59view0"}}

**Track:** [vLLM issue #62574](https://github.com/vllm-project/vllm/issues/62574)

### What changed

The two stacks expose opposite failure modes:

| Mixed-load case | Stack A: short TTFT / short decode | Stack B: short TTFT / short decode |
|---|---:|---:|
| 1 long + 1 short | 1.78 s / 3.6 tok/s | 1.61 s / 3.3 tok/s |
| 2 long + 2 short | 124.4 s / 21.1 tok/s | 3.03 s / 1.8 tok/s |
| 3 long + 2 short | 117.3 s / 16.9 tok/s | 9.22 s / 0.6 tok/s |
| 4 long + 2 short | 108.3 s / 19.4 tok/s | crash with 24 GB KV pin; 6.5 s / 0.5 tok/s with 16 GB |

Stack A largely queues the short requests until the long prefills finish. Stack B admits them in seconds, but their decode streams fall to 0.5–1.8 tokens/s. In the two- and three-prefill cases, Stack B reduces total short-request wall time by roughly 40 seconds, yet produces output slowly enough that the interactive experience remains poor. At three concurrent long prefills, its long-request TTFT also increases from approximately 120 seconds to 167 seconds. genui{"citation":{"ref":"turn59view0"}}

This distinguishes two kinds of starvation:

[
	ext{queue starvation}: quad T_{mathrm{TTFT}} gg 	ext{SLO}
]

[
	ext{execution starvation}: quad T_{mathrm{TTFT}} 	ext{is acceptable, but} T_{mathrm{POT}} gg 	ext{SLO}
]

A scheduler can improve the first while worsening the second. Reporting only TTFT would make Stack B appear far healthier than a user watching its 0.5-token/s stream would conclude.

### The second finding: KV allocation and workspace capacity are coupled

The patched stack exposes a nominal 3.23× maximum-context concurrency after pinning 24 GB for KV cache, versus 1.50× for Stack A. That extra advertised KV capacity does not translate into stable four-prefill execution.

With four concurrent long prefills, the 24 GB configuration reproducibly runs out of memory around 55K–58K computed tokens per request, followed by shared-memory broadcast stalls and engine failure. Reducing the KV pin to 16 GB avoids the crash, but long-prefill TTFT increases from about 111 seconds to 269 seconds. genui{"citation":{"ref":"turn59view0"}}

The practical capacity constraint is therefore not:

[
M_{mathrm{weights}} + M_{mathrm{KV}} leq M_{mathrm{device}}
]

but:

[
M_{mathrm{weights}} +
M_{mathrm{KV}} +
M_{mathrm{attention workspace}}(B,L) +
M_{mathrm{graphs}} +
M_{mathrm{runtime}}
leq
M_{mathrm{device}} - M_{mathrm{safety}}
]

where workspace demand changes with concurrent sequences, scheduled tokens, attention backend and context geometry. A larger manually pinned KV pool can reduce the headroom needed by the kernel that consumes it.

### Why this matters for CPU and EPYC inference

The exact measurements are GPU-specific, but the scheduling failure is not. On a CPU server, chunked prefill and decode contend for cores, memory controllers, LLC capacity and sometimes remote-socket bandwidth. A scheduler may admit decode every step yet give it too little effective bandwidth to meet an inter-token-latency SLO.

Token-count policies are especially fragile because equal token budgets do not imply equal execution time across:

- dense and MoE layers;
- full, sparse and recurrent attention;
- short and deep contexts;
- local and remote NUMA placement;
- GEMV-like and GEMM-like batch shapes;
- different quantization and kernel paths.

For an EPYC deployment, reserve service by measured resource time rather than by tokens alone. A workable controller would maintain recent estimates for prefill and decode cost, then enforce both a maximum prefill occupancy interval and a minimum decode service share:

[
rac{T_{mathrm{decode service}}}
{T_{mathrm{prefill service}}+T_{mathrm{decode service}}}
geq ho_{min}
]

subject to p99 TTFT and TPOT targets. If decode remains bandwidth-starved despite regular admission, isolate it to dedicated cores, CCDs or sockets, or move to prefill/decode disaggregation.

### Recommended regression test

Add an elephant/mouse workload to serving qualification:

1. Start one to four cache-cold long prefills.
2. Inject short requests at several offsets inside the prefill.
3. Sweep chunk size, maximum batched tokens, concurrency and cache allocation.
4. Repeat with scheduler and inference workers sharing cores, using static NUMA partitions, and using separated prefill/decode pools.

Record:

- p50/p99 TTFT and TPOT separately;
- worst streaming gap, not only average TPOT;
- long-prefill completion time;
- prefill and decode service time per scheduling window;
- memory bandwidth and remote-NUMA traffic by phase;
- KV occupancy, workspace high-water mark and allocator fragmentation;
- failure boundary as concurrent long contexts increase;
- completed requests meeting both TTFT and TPOT SLOs.

The last metric is the important one. Aggregate tokens/s can remain respectable while every interactive stream is degraded.

### Caveats

This report is unusually detailed, but it is not a controlled upstream benchmark:

- It compares different vLLM development versions and a vendor image against a heavily patched community recipe.
- Multiple settings differ, including KV allocation, graph capture, memory utilization and scheduler patches.
- It uses one model and a two-node GB10 unified-memory topology.
- The long prompts are synthetic filler designed to expose the failure boundary.
- No upstream fix or endorsed configuration exists yet.
- The OOM diagnosis is inferred from driver logs and similarity to an earlier workspace issue; a complete allocation trace has not yet isolated the responsible buffer.

The result should therefore motivate reproduction and instrumentation, not a general claim that prefill/decode interleaving is harmful.

## Assessment

This is a substantive deployment finding because it demonstrates two common benchmarking errors with concrete measurements:

1. **TTFT alone can hide decode starvation.**
2. **Nominal KV capacity can hide an execution-workspace cliff.**

For large-scale CPU inference, the immediate lesson is to schedule against time and shared-resource occupancy, then qualify capacity using the maximum simultaneous footprint—not a static post-load KV estimate. A mixed-workload server should be considered healthy only when it satisfies both first-token and ongoing-token SLOs without crossing its workspace safety margin.
