# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 2764.9 | 2765.0 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2526.9 | 2527.0 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2572.1 | 2572.1 |

Mean per stage (ms): embed **0.0** � retrieve **0.0** �
llm **2621.3** � total **2621.4**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

**N16 (Cloud/IaC):** stub — no cloud infrastructure actually deployed
**N17 (Data pipeline):** stub — no ETL jobs running  
**N18 (Lakehouse):** stub — no real data warehouse
**N19 (Vector + features):** stub — keyword fallback for embeddings, no real vector DB

**The dominant stage (llm, 100%) is exactly what I expected** because all other stages are stubbed. Embed=0ms and retrieve=0ms are the giveaway — a real RAG pipeline would spend 50-200ms on embedding and 10-50ms on retrieval before the LLM. Here the pipeline is just sending prompts to llama-server.

To halve pipeline latency, I would attack the LLM stage first — it is 100% of the latency. Options: (1) use a smaller/faster model, (2) enable prefix caching so repeated prompts skip prefill, or (3) batch multiple requests together. The other stages are negligible here because they are stubbed.
