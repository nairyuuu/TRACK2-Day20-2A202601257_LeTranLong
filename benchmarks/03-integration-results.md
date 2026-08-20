# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 807.4 | 807.5 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 527.9 | 527.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 529.2 | 529.3 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **621.5** · total **621.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

Component Status Declaration:
- N16 Cloud/IaC: Stub (local execution on Windows 11 workstation)
- N17 Data Pipeline: Stub (in-memory TOY_DOCS list)
- N18 Lakehouse: Stub (in-memory corpus)
- N19 Vector + Features: Stub (keyword lexical overlap fallback)
- N20 Serving: REAL (llama-server running Gemma 4 E2B on NVIDIA RTX 2060 with OpenAI-compatible API)

Analysis & Bottleneck Breakdown:
The dominant stage is llm (621.5 ms, 100.0% of total pipeline latency), while retrieval took only 0.1 ms. This matches architectural expectations for small in-memory corpus RAG, where auto-regressive token generation across 20-25 output tokens accounts for virtually all execution time.

To halve this pipeline\'s latency (2x speedup):
1. Speculative Decoding / MTP: Gemma 4 E2B supports Multi-Token Prediction (MTP). Leveraging a draft head or small draft model would reduce decode steps by ~1.8x.
2. Prompt Caching (Prefix Caching): Caching the shared system prompt and static RAG context via RadixAttention/prefix caching reduces prefill TTFT to near-zero.
