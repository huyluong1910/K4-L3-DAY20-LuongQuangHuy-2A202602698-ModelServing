# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` Â· llama.cpp `b10488` Â·
retrieval backend: **keyword overlap** Â· 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 7.3 | 8188.9 | 8196.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.2 | 7401.5 | 7401.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.2 | 7985.6 | 7986.0 |

Mean per stage (ms): embed **0.0** Â· retrieve **2.6** Â·
llm **7858.7** Â· total **7861.4**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 Cloud/IaC**: stub
- **N17 Data pipeline**: stub
- **N18 Lakehouse**: stub
- **N19 Vector + features**: stub (fallback dùng keyword overlap retrieval)
- **N20 Serving**: real (llama-server local)

Stage chiếm ưu thế hoàn toàn là llm (chiếm 100.0% tổng thời gian, trung bình 7858.7 ms so với retrieve chỉ 2.6 ms). Kết quả này hoàn toàn khớp với kỳ vọng lý thuyết: tìm kiếm văn bản trong bộ nhớ cực nhanh (chỉ vài ms), trong khi LLM inference trên CPU phải tính toán prefill hàng trăm token và autoregressive decode từng token một.
Nếu phải giảm độ trễ pipeline 2x, mục tiêu duy nhất cần tấn công là stage **llm**:
1. Áp dụng Prompt Caching / Prefix Caching để tái sử dụng KV cache của phần context đã retrieve, giảm mạnh TTFT.
2. Giới hạn độ dài output (max_tokens) hoặc dùng kỹ thuật Speculative Decoding.
3. Chuyển sang mô hình nhỏ hơn hoặc offload tính toán sang GPU chuyên dụng.
