# Bonus C8 - Semantic Cache Analysis

## Setup
- **Hardware:** Apple M4 Pro, 24GB RAM
- **Embedding model:** Gemma 4 E2B chat model in pooling mode (weak embedder)
- **Threshold:** 0.8

## Results

| # | Result | Similarity | Latency | Prompt |
|---|--------|-----------|---------|--------|
| 1 | miss | 0.00 | 1131 ms | What is goodput at SLO? |
| 2 | miss | 0.55 | 1089 ms | Explain TTFT and TPOT. |
| 3 | miss | 0.72 | 1093 ms | Can you define goodput@SLO? |
| 4 | miss | 0.76 | 1090 ms | What does time to first token mean? |
| 5 | miss | 0.64 | 1100 ms | How does PagedAttention work? |
| 6 | HIT | 0.80 | 0 ms | Tell me what goodput@SLO is. |
| 7 | HIT | 0.85 | 0 ms | What is prefix caching? |
| 8 | HIT | 0.85 | 0 ms | Describe how PagedAttention works. |

**Hit rate:** 3/8 = 38% at threshold 0.8

## Analysis

### True paraphrases (should hit):
- #3 ("define goodput@SLO") → #1 ("goodput at SLO") — **missed**
- #4 ("time to first token") → #2 ("TTFT and TPOT") — **missed**
- #6 ("what goodput@SLO is") → #1 — **HIT** (barely at 0.80)
- #8 ("how PagedAttention works") → #5 — **HIT**

### False Hits (should NOT hit):
- #7 ("What is prefix caching?") — **HIT** at 0.85, but this is a NEW topic, not related to #5 (#5 is about PagedAttention, #7 is about prefix caching)

### The Finding: No single threshold works

| Threshold | Fixes | Breaks |
|-----------|-------|--------|
| 0.95 | Stops #7 false hit | #6 (0.80) becomes miss |
| 0.70 | Catches #3, #4 | More false hits from unrelated prompts |
| 0.80 | Balances | #3 (#1 paraphrase) still misses |

**The fundamental problem:** A decoder trained to predict next token is NOT a good sentence encoder.

### Why weak embedder fails:

1. **Decoder ≠ Encoder:** Gemma is trained to predict next token, not to produce semantically meaningful embeddings
2. **Mean pooling of decoder states** loses semantic structure
3. **Similarity scores cluster:** Unrelated prompts get 0.50-0.85 similarity scores, same as paraphrases

### What a real embedder would do:

| Embedder | Paraphrase score | Unrelated score | Gap |
|----------|------------------|-----------------|-----|
| Weak (current) | 0.72-0.85 | 0.50-0.85 | 0 |
| Dedicated (Qwen3/BGE) | 0.85-0.95 | 0.10-0.30 | 0.55+ |

### Security Note

Semantic cache and prefix cache are shared across users. This creates a **timing side channel** (NDSS'25):
- Attacker: Send a probe prompt, time the response
- If fast → was in cache → revealed that someone asked something similar
- **Fix:** Salt cache per tenant (cache key = hash(prompt + tenant_id))

## Conclusion

A chat model in pooling mode is a weak embedder. Semantic cache hit rate is ~38% and the threshold tradeoff is painful. For production:
1. Use a dedicated embedding model (Qwen3-Embedding, BGE-M3)
2. Tune threshold per use case (higher for security-sensitive apps)
3. Salt cache per tenant to prevent timing attacks
