# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 784.0 | 784.0 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 448.9 | 448.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 466.9 | 466.9 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **566.6** · total **566.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Component | Status | Reason |
|-----------|--------|--------|
| N16 (llama-server) | **REAL** | Actual HTTP calls to localhost:8080, returned real answers |
| N17 (embed) | **STUB** | embed() returned None → fell back to keyword overlap |
| N18 (retrieve) | **STUB** | keyword overlap fallback, not real vector search |
| N19 (vector index) | **STUB** | TOY_DOCS hardcoded list, not real vector index |

**Is dominant stage what I expected?** Yes — LLM is 100% of pipeline time (566ms avg). The stubbed embed/retrieve stages took 0ms because they returned immediately without real computation.

**If I had to halve this pipeline's latency, I would attack:**

1. **LLM (prefill + decode)** — 100% of current time
   - Prefill is compute-bound: ~150ms for 110-150 tokens
   - Decode is memory-bandwidth-bound: ~290ms for 22-24 tokens
   - To halve: reduce context length (fewer retrieved docs) or use faster quantization

2. **Real embed/retrieve (if upgraded)** — would become non-trivial
   - A real embedding model (e.g., nomic-embed-text) adds 50-200ms
   - A real vector DB (Qdrant/Milvus) adds 10-50ms
   - With real components, embed+retrieve could be 10-20% of total, not 0%
