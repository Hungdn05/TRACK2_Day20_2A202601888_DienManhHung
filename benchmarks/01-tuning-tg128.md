# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **12 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 83.8 | 100% |
| 6 | 84.2 | 100% |
| 12 | 79.1 | 94% |
| 24 | 70.9 | 84% |

**Best**: `-t 6` at 84.2 tok/s
**Slowest tested**: `-t 24` at 70.9 tok/s (1.19x spread)
**Against the physical-core default** (`-t 12`, 79.1 tok/s): 1.06x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation (required -- replace this line)

_Where is the knee, and why there? If the peak sits at your physical core count
and drops above it, say what the extra threads are competing for. If your curve
does something else -- flat, or still climbing at 2x logical cores -- say that
instead and reason about why. A result that contradicts the expected shape is
worth more than one that matches it, as long as you explain it._
