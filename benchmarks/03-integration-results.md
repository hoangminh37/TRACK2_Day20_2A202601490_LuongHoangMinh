# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 980.5 | 980.5 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 727.2 | 727.2 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 729.8 | 729.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **812.5** · total **812.5**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **Khai báo Real / Stub:**
  - **N16 Cloud/IaC:** *stub* (chạy hoàn toàn local trên laptop).
  - **N17 Data pipeline:** *stub* (sử dụng tập dữ liệu toy docs in-memory).
  - **N18 Lakehouse:** *stub* (danh sách Python object cố định).
  - **N19 Vector + features:** *stub* (retrieval bằng thuật toán keyword overlap đơn giản, chưa tích hợp vector DB).
  - **N20 Serving:** **real** (kết nối trực tiếp tới `llama-server` qua HTTP streaming endpoint `/v1/chat/completions`).
- **Phân tích Bottleneck & Tối ưu:**
  - **Stage chiếm nhiều nhất:** `llm` chiếm **100%** tổng thời gian (812.5 ms trên tổng 812.5 ms), trong khi `embed` và `retrieve` gần như bằng 0.0 ms do chạy stub trên bộ nhớ. Kết quả này hoàn toàn khớp với kỳ vọng vì suy luận sinh token của LLM luôn là phần tốn kém tài nguyên tính toán và băng thông nhất trong pipeline RAG.
  - **Phương án giảm latency 2×:** Cần tấn công trực tiếp vào stage **LLM**:
    1. **Tối ưu TTFT (Prefill):** Bật Prompt Caching (RadixAttention / Prefix cache) để tái sử dụng KV cache của System Prompt và context tài liệu chung giữa các query.
    2. **Tối ưu TPOT (Decode):** Áp dụng Speculative Decoding (MTP head) hoặc sử dụng thread tuning tối ưu (`-t 5`) và model quant gọn hơn để đẩy tốc độ decode lên cao nhất.
