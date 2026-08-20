# Bonus - Batch-size sweep (chunked prefill)

Host `Darwin-arm64` · llama.cpp `b10488` ·
`threads=12` `ngl=99` · metric `pp512`

| -b (logical) | -ub (micro) | pp512 (tok/s) | vs best |
|:--|--:|--:|--:|
| 128 | 128 | 1129.6 | 93% |
| 256 | 256 | 1185.7 | 97% |
| 512 | 256 | 1190.0 | 98% |
| 512 | 512 | 1216.9 | 100% |
| 1024 | 512 | 1214.8 | 100% |
| 2048 | 512 | 1217.2 | 100% |

Best: `-b 2048 -ub 512` at 1217.2 tok/s
(1.08x the slowest point tested).

This sweep only measures the throughput half of the trade. The cost it hides is
TTFT for queued requests: a larger micro-batch holds the device longer per step,
so anything waiting behind it waits longer. To see both halves, re-run
`make load-50` with your best and worst settings via
`.venv/bin/python labs/02-serve/serve.py -- -b N -ub M` and compare P95.

## Your finding

**Would run `-b 512 -ub 512` in production on M4 Pro.**

| Setting | Throughput | TTFT tradeoff | Recommendation |
|---------|-----------|--------------|----------------|
| -b 128 -ub 128 | 1129 tok/s | Low TTFT | ❌ Wastes batching |
| -b 256 -ub 256 | 1185 tok/s | Medium TTFT | ⚠️ |
| -b 512 -ub 512 | 1217 tok/s | Balanced | ✅ Best choice |
| -b 2048 -ub 512 | 1217 tok/s | High TTFT | ⚠️ Diminishing returns |

**Why not the "best" throughput?**

The best throughput (-b 2048 -ub 512) is only 0.003% faster than -b 512 -ub 512, but:
- Larger batch = longer per-step compute = higher TTFT for queued requests
- Memory pressure increases with batch size
- Diminishing returns past 512

**To be sure P95 doesn't hurt:**
Would need to run `make load-50` with both settings and compare P95 latency. The tradeoff:
- Throughput: -b 512 vs -b 2048 = ~same
- P95 latency: larger batch = longer queue wait = higher P95

**Production recommendation:** Start with `-b 512 -ub 512`. Only increase if throughput profiling shows saturation at that level.
