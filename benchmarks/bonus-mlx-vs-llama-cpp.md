# Bonus B5 - MLX vs llama.cpp Metal

Host `Darwin-arm64` · arm ·
llama.cpp `b10488` · 10 prompts,
`max_tokens=64`, warm-up discarded on both sides

| Runtime | Weights | TTFT P50 (ms) | TTFT P95 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|
| llama.cpp (Metal) | `gemma-4-E2B-it-UD-Q4_K_XL.gguf` | 116.0 | 135.8 | 46.7 |
| MLX-LM | `unsloth/gemma-4-E2B-it-UD-MLX-4bit` | 87.7 | 194.5 | 53.8 |

MLX decode is **1.15x** llama.cpp Metal here. Faster on decode: **MLX**.

Both sides run the same model at 4-bit and stream token by token, so the gap is a
runtime difference, not a model difference. The quantization schemes are not
byte-identical, though (Unsloth Dynamic GGUF vs MLX 4-bit), so treat a gap under
~10% as noise rather than a finding.

## Your finding

- **Lựa chọn triển khai (Shipping decision):** Tôi sẽ chọn **llama.cpp**.
- **Lý do (Why):** Mặc dù MLX đem lại tốc độ decode nhanh hơn ~15% (53.8 vs 46.7 tok/s) và TTFT P50 thấp hơn (87.7 vs 116.0 ms), nhưng có một số nhược điểm lớn khi đưa vào thực tế:
  1. **Triển khai (Deployment):** `llama.cpp` đóng gói toàn bộ inference engine vào một file binary duy nhất không phụ thuộc vào các thư viện Python phức tạp, cực kỳ dễ đóng gói và triển khai.
  2. **Tính di động đa nền tảng (Portability):** `llama.cpp` hỗ trợ đa nền tảng (Apple Metal, NVIDIA CUDA, Vulkan). Nếu dự án sau này mở rộng quy mô và cần chuyển khỏi máy tính Mac sang server NVIDIA ở Data Center, hệ thống dùng `llama.cpp` vẫn hoạt động bình thường, trong khi mã nguồn dùng `MLX` sẽ phải viết lại từ đầu do MLX bị "khoá chặt" (vendor-lock) vào chip Apple Silicon.
  3. **Độ trễ đuôi (Tail Latency):** TTFT P95 của MLX (194.5 ms) cao hơn hẳn `llama.cpp` (135.8 ms), cho thấy `llama.cpp` ổn định và đáng tin cậy hơn trong môi trường production có biến động tải.
