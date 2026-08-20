# 02 - Continuous batching under load (u50)

Host `Darwin-arm64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.96 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 8001 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

- **Peak batch width đạt 3.96 trên tổng số 4 slots (~99% công suất):** Con số này chứng minh thuật toán Continuous Batching (iteration-level batching) của `llama-server` hoạt động cực kỳ hiệu quả dưới tải cao, liên tục gom các requests đồng thời vào cùng một bước tính toán decode trên GPU/Metal.
- **So sánh với Effective Concurrency (31.9):** 
  - Gauge `n_busy_slots_per_decode` (3.96) phản ánh đúng **Compute Utilization (Slot Utilisation)** thực tế của phần cứng tại từng bước giải mã (bị giới hạn tối đa bởi `--parallel 4`).
  - Trong khi đó, Effective Concurrency (31.9) từ Little's Law phản ánh **Traffic Pressure / System Occupancy** (tổng số request đang nằm trong hệ thống). Sự chênh lệch (31.9 so với ~4) là do có tới 46 request bị hoãn lại (`requests_deferred`) và phải xếp hàng chờ trong queue.
  - Cả hai con số đều đáng tin cậy và bổ trợ lẫn nhau: một bên đo tải tính toán thực tế, một bên đo áp lực hàng đợi.
