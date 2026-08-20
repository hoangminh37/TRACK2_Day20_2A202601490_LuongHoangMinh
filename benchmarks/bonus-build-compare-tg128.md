# Bonus B1 - Prebuilt vs source build

Host `Darwin-arm64` · CPU `Apple M4`
Vector extensions detected: NEON
llama.cpp `b10488` both sides · `threads=10` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 18.6 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 24.1 | 1.29x |

On this machine, the source build is **1.29x faster**.

before: 18.6 tok/s (prebuilt release)
after:  24.1 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 1.29x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.


### Separately: what GPU offload is worth on the same binary

`tg128` on the source build at `-ngl 99` instead of `-ngl 0`:

| Source build | tg128 (tok/s) | vs its own CPU run |
|:--|--:|--:|
| `-ngl 0` (CPU) | 24.1 | 1.00x |
| `-ngl 99` (offloaded to MTL0: Apple M4 (12124 MiB, 12123 MiB free)) | 42.8 | 1.78x |

This number is **not** part of the B1 comparison above -- it is a different knob.
Reporting it separately is the point: a compiler flag and an accelerator are not
interchangeable explanations for a speedup.


## Your explanation

Sự chênh lệch tốc độ 1.29x (24.1 tok/s so với 18.6 tok/s) đến từ việc biên dịch native:
1. **Lợi thế Compile Native (`-DGGML_NATIVE=ON`):** Bản prebuilt phải được build ở cấu hình an toàn (baseline ARM64 chung) để đảm bảo tính tương thích và có thể chạy trên nhiều đời chip Apple Silicon cũ hơn. Do đó, nó không thể tận dụng tối đa các tập lệnh tối ưu chuyên biệt. Trong khi đó, bản tự build từ mã nguồn đã cho phép trình biên dịch (compiler) nhận diện chính xác kiến trúc phần cứng hiện tại (Apple M4) và khai thác triệt để bộ tập lệnh mở rộng (NEON / ARMv9) tiên tiến nhất trên con chip này để vector hoá các phép tính ma trận.
2. **CPU Bound vs Memory Bound:** Khi ta ép chạy trên CPU (`-ngl 0`), việc decode không chỉ phụ thuộc băng thông mà còn bị giới hạn bởi năng lực tính toán (Compute-bound) của CPU. Lúc này, các tập lệnh vector NEON tối ưu đã giúp CPU xử lý số lượng phép tính lớn trong 1 chu kỳ nhanh hơn hẳn, đem lại tốc độ 1.29x. Khi bật offload sang Metal (`-ngl 99`), tốc độ nhảy vọt lên 42.8 tok/s (1.78x) do Metal xử lý tính toán song song tốt hơn và băng thông bộ nhớ tới GPU nhanh hơn.
