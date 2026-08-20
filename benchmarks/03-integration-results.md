# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 6697.5 | 6697.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5629.5 | 5629.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 5811.7 | 5811.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **6046.2** · total **6046.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Các thành phần real và stub

- N16 Cloud/IaC: stub, chạy localhost thay cho cluster/Compose.
- N17 Data pipeline: stub, dùng danh sách trong bộ nhớ.
- N18 Lakehouse: stub, dùng toy dictionary thay cho Delta/Iceberg.
- N19 Vector + features: stub, `TOY_DOCS` và keyword overlap thay cho vector index.
- N20 Serving: real, request đi qua `llama-server` OpenAI-compatible.

LLM chiếm 6046.2/6046.3 ms, xấp xỉ 100%, đúng kỳ vọng vì embed bị tắt và retrieval toy
chỉ mất 0.1 ms. Muốn giảm tổng latency 2x, tôi sẽ tối ưu LLM decode trước: dùng Q2,
6 threads và giảm output budget; tối ưu retrieval gần như không thay đổi tổng thời gian.
