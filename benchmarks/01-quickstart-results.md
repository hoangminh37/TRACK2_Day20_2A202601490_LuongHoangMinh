# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3083 | 120 / 196 | 20.7 / 21.2 | 1404 / 1501 / 1501 | 48.4 |
| UD-Q2_K_XL | 2.24 | 3043 | 117 / 316 | 16.9 / 17.2 | 1187 / 1376 / 1376 | 59.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.22x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

- **Về tốc độ và tài nguyên:** Bản `UD-Q2_K_XL` (2.24 GB) giảm ~24.6% dung lượng bộ nhớ so với `UD-Q4_K_XL` (2.97 GB). Do giai đoạn decode bị giới hạn bởi băng thông bộ nhớ (Memory Bandwidth bound), việc giảm kích thước weight giúp TPOT P50 giảm từ 20.7 ms xuống 16.9 ms, mang lại tốc độ decode nhanh hơn **1.22×** (59.0 tok/s so với 48.4 tok/s). TTFT P50 ở 2 bản tương đương nhau (~117–120 ms).
- **Về chất lượng và tính thực tế:** Nhờ kỹ thuật Unsloth Dynamic (UD) giữ độ chính xác cao ở các layer nhạy cảm, bản 2-bit vẫn trả lời mạch lạc các câu hỏi thông thường. Tuy nhiên, với các câu hỏi lập luận phức tạp hoặc yêu cầu định dạng chặt chẽ, bản 4-bit (`UD-Q4_K_XL`) cho câu trả lời chuẩn xác và chi tiết hơn rõ rệt.
- **Kết luận đánh đổi:** Trên máy Apple Silicon hiện tại, tốc độ 48.4 tok/s của bản 4-bit đã rất mượt mà và vượt xa tốc độ đọc của người dùng. Do đó, việc đánh đổi độ chính xác của mô hình để lấy thêm ~10.6 tok/s ở 2-bit là **không thực sự đáng**, trừ khi hệ thống bị giới hạn nghiêm ngặt về dung lượng RAM/VRAM.
