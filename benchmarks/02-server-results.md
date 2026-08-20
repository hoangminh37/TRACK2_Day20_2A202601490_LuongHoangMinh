# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=10` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 70 | 1.24 | 6700 | 9300 | 10000 | 8.2 | 0.0% |
| 50 | 68 | 1.16 | 30000 | 45000 | 48000 | 31.9 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.94x** (19% of linear) |
| P95 latency | **4.84x** |
| Effective concurrency at 50 users | 31.9 vs `--parallel 4` slots (occupancy/slot ratio 7.98) |

**Saturated.** Throughput delivered only 0.94x for 5x the offered load, and effective concurrency (31.9) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.94x while P95 moved 4.84x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

- **Điểm bão hoà (Saturation point):** Server bão hoà hoàn toàn ở mức ≤ 50 users (thực tế đã bắt đầu bão hòa ngay từ mốc ~10–15 users khi 4 slots xử lý bị lấp đầy).
- **Bằng chứng thuyết phục:**
  - Khi tăng tải mô phỏng gấp 5× (từ 10 lên 50 users), thông lượng thực tế (**RPS**) không hề tăng mà giảm nhẹ từ 1.24 xuống 1.16 req/s (**0.94×**).
  - Trong khi đó, **P95 latency tăng vọt 4.84×** (từ 9.3s lên 45.0s) và **P50 tăng từ 6.7s lên 30.0s**. Theo Little's Law, Effective Concurrency đạt **31.9**, vượt gấp 8 lần sức chứa của 4 slots decode (`--parallel 4`). Phần latency tăng thêm (~35.7s ở P95) hoàn toàn là **Queue time** (chờ trong hàng đợi `requests_deferred` lên tới 46 request), không phải compute time.
- **Chiến lược nâng Goodput@SLO:**
  - Nếu đặt mục tiêu SLO là P95 ≤ 10s: Ở 10 users, toàn bộ 1.24 RPS đều đạt SLO (Goodput = 1.24 RPS). Ở 50 users, Goodput@SLO sụt giảm nghiêm trọng về gần 0 vì P95 (45s) đã vi phạm SLO.
  - **Knob sẽ thay đổi đầu tiên:** Áp dụng **Request Admission Control / Queue Shedding** (từ chối hoặc trả về 429 khi queue đầy) kết hợp tăng `--parallel` (nếu dung lượng Unified Memory cho phép mở rộng KV cache). Điều này giúp giữ P95 trong ngưỡng SLO, ngăn chặn queue-collapse và duy trì goodput tối đa cho hệ thống.
