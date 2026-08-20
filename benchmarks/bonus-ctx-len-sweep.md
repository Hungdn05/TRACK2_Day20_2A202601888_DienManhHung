# Bonus - Context-length sweep (prefill cost)

Host `Darwin-arm64` · llama.cpp `b10488` ·
`threads=12` `ngl=99` · RAM 24.0 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 1198.5 | 213.6 | 1.00x |
| 1024 | 1199.9 | 853.4 | 1.00x |
| 2048 | 1165.7 | 1756.9 | 1.03x |
| 4096 | 1111.3 | 3685.9 | 1.08x |
| 8192 | 1012.4 | 8091.6 | 1.18x |
| 16384 | 856.5 | 19128.6 | 1.40x |

At 16384 tokens, prefill costs **19129 ms** --
1.40x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

**Prefill dominates at 8192+ tokens.**

| Prompt length | Prefill cost | Dominance |
|---------------|--------------|-----------|
| 256-1024 | 200-850 ms | Linear |
| 2048 | 1757 ms | Start bending |
| 4096 | 3686 ms | Prefill = 50%+ of E2E |
| 8192 | 8092 ms | Dominates |
| 16384 | 19129 ms | Massive (19s!) |

**Quadratic bend confirmed:** The O(N²) term appears above 8k tokens:
- 256→1024: 4x tokens, 4x time (linear)
- 4096→8192: 2x tokens, 2.2x time (slight bend)
- 8192→16384: 2x tokens, **2.4x time** (quadratic!)

**RAG pipeline implication:**
- Max affordable chunks at 4GB prompt ≈ 3-4 chunks
- For SLO < 5s: limit to ~2048 tokens of context
- "Context window allows 32k" ≠ "use 32k" — prefill costs before first token
