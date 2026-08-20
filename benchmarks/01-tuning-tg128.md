# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **12 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 82.8 | 98% |
| 6 | 84.1 | 100% |
| 12 | 79.1 | 94% |
| 24 | 68.5 | 81% |

**Best**: `-t 6` at 84.1 tok/s
**Slowest tested**: `-t 24` at 68.5 tok/s (1.23x spread)
**Against the physical-core default** (`-t 12`, 79.1 tok/s): 1.06x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

**Knee is at 6 threads, NOT at physical core count (12).**

| threads | tok/s | vs best | Observation |
|---------|-------|---------|-------------|
| 1 | 82.8 | 98% | baseline |
| 6 | 84.1 | **100%** | **BEST** |
| 12 | 79.1 | 94% | oversubscribed vs 6 |
| 24 | 68.5 | 81% | severe oversubscription |

**Why the peak at 6, not 12?**

The M4 Pro has 12 physical performance cores, but the decode phase (tg128) is **memory-bandwidth bound**, not compute-bound. Each decode token requires reading the KV cache + weights from unified memory. At 6 threads, the memory bandwidth is saturated but not over-subscribed.

1. **Memory bandwidth saturation**: With `ngl=99` (full GPU offload to Metal), the actual compute is happening on the GPU. The CPU threads feed prompts and collect results — they don't do the heavy KV matmul. Extra CPU threads beyond 6 hit diminishing returns on feeding the GPU.

2. **Metal GPU coordination overhead**: More threads = more synchronization overhead coordinating with the Metal GPU. The GPU runs asynchronously; having 12 CPU threads all trying to manage GPU work creates contention.

3. **Cache thrashing**: With 12 threads all reading model weights from L2 cache, cache eviction increases. 6 threads fit better in cache, reducing memory traffic.

4. **Diminishing returns past peak**: Going from 12→24 threads drops 19% (from 94% to 81%). This confirms memory bandwidth is the ceiling — extra threads spend more time waiting for RAM than doing useful work.

**Conclusion**: The expected "peak at physical core count" heuristic holds for compute-bound workloads. Decode is memory-bandwidth-bound, so the optimal thread count is lower — in this case, 6 threads (half the physical core count).
