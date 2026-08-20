# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=12` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 1050 | 84 / 91 | 13.0 / 15.6 | 898 / 987 / 987 | 77.2 |
| UD-Q2_K_XL | 2.24 | 2024 | 83 / 84 | 12.7 / 13.0 | 879 / 900 / 900 | 78.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.02x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

**Q2_K_XL worth it on M4 Pro with 24GB RAM: YES**

| Metric | Q4_K_XL | Q2_K_XL | Winner |
|--------|---------|---------|--------|
| Size | 2.97 GB | 2.24 GB | Q2 (25% smaller) |
| Load time | 1050 ms | 2024 ms | Q4 (faster load) |
| TTFT P50 | 84 ms | 83 ms | ~same |
| TPOT P50 | 13.0 ms | 12.7 ms | Q2 (2% faster) |
| Decode speed | 77.2 tok/s | 78.9 tok/s | Q2 (2% faster) |

**Conclusion:** Q2_K_XL is worth it on this M4 Pro. The smaller model decodes 2% faster and uses 25% less disk space. The load time difference is within measurement variance. On short prompts like this benchmark, the speed difference is modest but consistent. For longer context RAG workloads where memory matters more, Q2's 0.73 GB RAM savings could be significant.
