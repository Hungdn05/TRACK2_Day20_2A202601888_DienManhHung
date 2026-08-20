# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=12` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 102 | 1.81 | 4400 | 6400 | 7500 | 8.1 | 0.0% |
| 50 | 105 | 1.78 | 27000 | 29000 | 30000 | 37.1 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.99x** (20% of linear) |
| P95 latency | **4.53x** |
| Effective concurrency at 50 users | 37.1 vs `--parallel 4` slots (occupancy/slot ratio 9.29) |

**Saturated.** Throughput delivered only 0.99x for 5x the offered load, and effective concurrency (37.1) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.99x while P95 moved 4.53x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Saturation point: between 10 and 50 users**

Evidence:

| Metric | 10 users → 50 users | Interpretation |
|--------|---------------------|----------------|
| Offered load | 5x | |
| Throughput (RPS) | 1.81 → 1.78 (0.99x) | **Plateaued** — no scaling |
| P95 latency | 6400ms → 29000ms (4.53x) | **Exploded** — queueing |
| Eff. concurrency | 8.1 → 37.1 | Requests accumulating |
| Slot usage | ~2 slots avg | 4 available |

**The number that convinced me:** Effective concurrency = 37.1 while RPS stayed flat at ~1.78. This means 37 concurrent requests were in flight but only ~1.78 completed per second. The server was processing 4 requests in parallel (batching) but 33+ were waiting in queue. Throughput hit the ceiling; extra users only added wait time.

**First knob to raise goodput:** `--parallel` (slot count)

Why `--parallel` and not thread count?
1. The batching data shows 3.96/4 slots busy — **slots are the bottleneck**, not CPU threads
2. Increasing threads (tune results) gives marginal gains (~6%) because decode is memory-bandwidth bound
3. Increasing `--parallel` from 4 to 8 doubles batching capacity — each decode step processes 2x requests
4. Memory is sufficient (24GB RAM >> 3GB model)

**Secondary consideration:** If P95 SLO is <10s, even 8 slots may not help — the underlying TPOT (~13ms) sets a floor. The bottleneck then is memory bandwidth per token, not queue length.
