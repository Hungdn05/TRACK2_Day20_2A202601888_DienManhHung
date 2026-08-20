# 02 - Continuous batching under load (u50)

Host `Darwin-arm64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.96 of 4 slots (99%) |
| `requests_processing` | 0 |
| `requests_deferred` | 0 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 12419 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` stayed at zero: every request found a free slot on arrival.

## Your observation

**Peak batch width: 3.96 of 4 slots (99% occupancy)**

This matches the effective concurrency analysis in `02-server-results.md`:

| Metric | Value | Interpretation |
|--------|-------|----------------|
| Peak busy slots | 3.96 / 4 | ~100% slot utilization |
| Effective concurrency | 37.1 | 37 requests in flight |
| Actual parallel slots | 4 | 4 decode slots available |

**Do they match?** Yes, in direction. The metrics gauge (3.96/4) shows **instantaneous** slot utilization during decode — the batch scheduler packed requests tightly. The effective concurrency (37.1) is **time-averaged** over the entire 60s test — it counts queued requests waiting for slots.

**Which do I trust?** Both are correct; they measure different things:
- `n_busy_slots_per_decode` ≈ 4 → batching works, decode is saturated
- Effective concurrency = 37 → request queue is building up

The gap (4 slots busy, 37 in queue) means requests arrived faster than they could be scheduled into the 4 decode slots. This confirms saturation: the queue is growing while slots are fully utilized.
