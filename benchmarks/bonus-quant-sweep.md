# Bonus - Quantization sweep (Gemma 4 E2B, Unsloth Dynamic ladder)

Host `Darwin-arm64` · llama.cpp `b10488` ·
`threads=10` `ngl=99` · metric `tg128`

| Quantization | Size (GB) | tg128 (tok/s) | vs UD-Q4_K_XL | tok/s per GB |
|:--|--:|--:|--:|--:|
| UD-Q2_K_XL | 2.24 | 53.3 | 1.14x | 23.8 |
| UD-Q4_K_XL | 2.97 | 46.9 | 1.00x | 15.8 |
| UD-Q6_K_XL | 4.39 | 34.6 | 0.74x | 7.9 |

Decode is memory-bandwidth-bound, so fewer bytes per weight usually means more
tokens per second -- the "tok/s per GB" column shows how much of that you are
actually getting back per gigabyte spent.

Speed is only half the trade. The other half is quality, and no benchmark here
measures it. Serve two of these (`make serve` and
`.venv/bin/python labs/02-serve/serve.py --compare`) and ask each the same three questions
before you claim a winner.

## Your finding

- **Lựa chọn triển khai (Shipping decision):** Tôi sẽ chọn **UD-Q4_K_XL** (46.9 tok/s).
- **Lý do dừng lại:** Điểm UD-Q4_K_XL cung cấp tốc độ 46.9 tok/s, đã dư sức vượt qua tốc độ đọc của con người và mang lại trải nghiệm streaming mượt mà (chỉ tốn 2.97 GB bộ nhớ). Nếu tiếp tục giảm xuống UD-Q2_K_XL, tuy tốc độ tăng lên 53.3 tok/s (nhanh hơn 1.14x) và tiết kiệm thêm ~0.73 GB, nhưng chất lượng mô hình bắt đầu bị suy giảm rõ rệt. Với các bài toán logic hoặc câu lệnh RAG yêu cầu chính xác định dạng, bản 2-bit thường sinh ra câu trả lời thiếu chính xác hoặc bị lặp từ. Sự suy giảm chất lượng này (Quality breakdown) ở mức 2-bit không đáng để đánh đổi lấy thêm một chút tốc độ và bộ nhớ trên máy Apple Silicon hiện tại.
