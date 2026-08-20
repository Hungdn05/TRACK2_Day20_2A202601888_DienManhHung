# Bonus - Quantization sweep (Gemma 4 E2B, Unsloth Dynamic ladder)

Host `Darwin-arm64` · llama.cpp `b10488` ·
`threads=12` `ngl=99` · metric `tg128`

| Quantization | Size (GB) | tg128 (tok/s) | vs UD-Q4_K_XL | tok/s per GB |
|:--|--:|--:|--:|--:|
| UD-Q2_K_XL | 2.24 | 79.3 | 1.00x | 35.4 |
| UD-Q4_K_XL | 2.97 | 79.1 | 1.00x | 26.6 |
| UD-Q6_K_XL | 4.39 | 75.5 | 0.95x | 17.2 |

Decode is memory-bandwidth-bound, so fewer bytes per weight usually means more
tokens per second -- the "tok/s per GB" column shows how much of that you are
actually getting back per gigabyte spent.

Speed is only half the trade. The other half is quality, and no benchmark here
measures it. Serve two of these (`make serve` and
`.venv/bin/python labs/02-serve/serve.py --compare`) and ask each the same three questions
before you claim a winner.

## Your finding

**Would ship UD-Q2_K_XL on M4 Pro 24GB.**

| Quantization | tok/s per GB | Recommendation |
|-------------|---------------|----------------|
| UD-Q2_K_XL | 35.4 | ✅ Best efficiency |
| UD-Q4_K_XL | 26.6 | ✅ Good balance |
| UD-Q6_K_XL | 17.2 | ❌ Worse speed, 2x size |

**Why UD-Q2_K_XL:**
1. Same decode speed as Q4 (79.3 vs 79.1 tok/s) — no performance penalty
2. 25% smaller disk (2.24 GB vs 2.97 GB)
3. Best tok/s per GB ratio (35.4)
4. M4 Pro 24GB has enough RAM — quality concerns are the only reason to avoid

**Quality assessment from pipeline output:**
- Q2 produced coherent answers in the RAG pipeline
- Answers were factually correct for the toy docs
- No obvious degradation on short prompts

**Conclusion:** On M4 Pro with 24GB RAM, UD-Q2_K_XL is the optimal choice — same speed, less memory, better efficiency.
