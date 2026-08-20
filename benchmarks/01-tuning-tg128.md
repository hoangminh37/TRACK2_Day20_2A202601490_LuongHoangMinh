# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **10 physical · 10 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 51.1 | 98% |
| 5 | 52.0 | 100% |
| 10 | 49.5 | 95% |
| 20 | 40.0 | 77% |

**Best**: `-t 5` at 52.0 tok/s
**Slowest tested**: `-t 20` at 40.0 tok/s (1.30x spread)
**Against the physical-core default** (`-t 10`, 49.5 tok/s): 1.05x

Use this in your run:

```bash
LAB_N_THREADS=5 make bench
```

## Your explanation

- **Vị trí điểm "Knee" (Điểm tối ưu):** Điểm tối ưu đạt đỉnh tại **`-t 5`** với tốc độ **52.0 tok/s** (vượt qua mức mặc định 10 cores là 49.5 tok/s, mang lại tỉ lệ 1.05×).
- **Cơ chế tại sao `-t 5` đạt hiệu năng cao nhất:**
  - Trên kiến trúc Apple Silicon (10 physical cores gồm cụm P-cores hiệu năng cao và E-cores tiết kiệm điện), việc đặt 5 threads khớp với số lượng P-cores chính, tận dụng tối đa năng lực tính toán và băng thông L2 cache mà không bị kéo chậm bởi các nhân E-cores (tránh hiện tượng *straggler effect* khi các thread phải chờ barrier synchronization giữa P-core nhanh và E-core chậm).
  - Khi đã bật Metal GPU offload (`ngl=99`), quá trình autoregressive decode bị giới hạn bởi Unified Memory Bandwidth. Do đó chỉ cần một lượng nhỏ thread (1–5 threads) để dispatch lệnh sang GPU/NEON là đã bão hòa băng thông đọc weights.
- **Hiện tượng suy giảm khi tăng threads (`-t 10` và `-t 20`):**
  - **Ở `-t 10` (49.5 tok/s):** Hệ điều hành phân phối thread sang cả các nhân E-cores, làm tăng độ trễ đồng bộ (synchronization overhead) và gây tranh chấp tài nguyên cache L2/băng thông bộ nhớ.
  - **Ở `-t 20` (40.0 tok/s - giảm ~23% so với đỉnh):** Hiện tượng *thread oversubscription* (20 threads tranh chấp trên 10 core vật lý) gây ra chi phí chuyển đổi ngữ cảnh liên tục (context switching overhead) và cache thrashing nghiêm trọng, làm giảm đáng kể thông lượng sinh token.
