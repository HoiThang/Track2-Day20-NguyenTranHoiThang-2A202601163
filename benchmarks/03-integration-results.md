# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 2937.5 | 2937.5 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2794.0 | 2794.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 2741.3 | 2741.4 |

Mean per stage (ms): embed **0.0** � retrieve **0.0** �
llm **2824.3** � total **2824.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

**N16 (Cloud/IaC):** stub — no cloud infrastructure actually deployed
**N17 (Data pipeline):** stub — no ETL jobs running
**N18 (Lakehouse):** stub — no real data warehouse
**N19 (Vector + features):** stub — keyword fallback for embeddings, no real vector DB

**The dominant stage (llm, 100%) is exactly what I expected** because all other stages are stubbed. In a real production RAG system, embed would be 50-200ms and retrieve 10-50ms, making llm still dominant but not 100%. To halve pipeline latency, I would attack the LLM stage first — use a smaller/faster model or enable KV cache / prefix caching for repeated queries.
