# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Lương Hoàng Minh
**Cohort:** AICB-P2T2 (Track 2)
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** macOS 15 (Darwin 25.5.0)
- **CPU:** Apple M4
- **Cores:** 10 physical / 10 logical
- **CPU extensions:** NEON
- **RAM:** 16.0 GB
- **Accelerator:** Apple Metal
- **llama.cpp asset đã tải:** llama-b10488-bin-macos-arm64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Setup diễn ra mượt mà nhờ script tự động tải prebuilt release binary b10488 tích hợp sẵn backend Metal cho Apple Silicon và tải bộ weights Gemma 4 E2B mà không cần compile.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3083 | 120 / 196 | 20.7 / 21.2 | 1404 / 1501 / 1501 | 48.4 |
| UD-Q2_K_XL | 2.24 | 3043 | 117 / 316 | 16.9 / 17.2 | 1187 / 1376 / 1376 | 59.0 |

**Quan sát** (≤ 60 chữ): Bản 2-bit decode nhanh hơn 1.22× (59.0 vs 48.4 tok/s) do giảm tải băng thông bộ nhớ. Tuy nhiên, bản 4-bit giữ được lập luận logic và format tốt hơn; với tốc độ 48.4 tok/s đã rất mượt thì sự đánh đổi lấy 2-bit là không cần thiết.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.24 | 6700 | 9300 | 10000 | 8.2 | 0.0% |
| 50 | 1.16 | 30000 | 45000 | 48000 | 31.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.94×
- **P95 tăng:** 4.84×
- **Effective concurrency ở 50 users:** 31.9 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.96 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà ở ≤ 50 users (từ ~10–15 users). Bằng chứng: RPS không tăng (0.94×) nhưng P95 tăng 4.84× (9.3s lên 45s), Eff. concurrency đạt 31.9. Độ trễ tăng là Queue time do `requests_deferred` = 46. Để nâng Goodput@SLO (P95 ≤ 10s), tôi sẽ áp dụng Admission Control (Queue Shedding) kết hợp tăng `--parallel`.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local macOS runtime | stub |
| N17 Data pipeline | In-memory toy docs | stub |
| N18 Lakehouse | In-memory Python list | stub |
| N19 Vector + features | Keyword overlap fallback | stub |
| N20 Serving | `llama-server` (:8080) | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 812.5 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck nằm 100% ở LLM inference (812.5 ms) đúng như kỳ vọng vì embed/retrieve chạy in-memory stub. Để giảm latency 2×, tôi sẽ tối ưu prefill bằng Prompt Caching (RadixAttention) và tối ưu decode bằng Speculative Decoding (MTP head).

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tối ưu hóa số lượng luồng thực thi (Thread count tuning): Chuyển từ mặc định 10 physical cores (`-t 10`) sang điểm tối ưu 5 P-cores (`-t 5`), đồng thời tránh oversubscription (`-t 20`).

```
before:  49.5 tok/s (-t 10) / 40.0 tok/s (-t 20)
after:   52.0 tok/s (-t 5)
speedup: 1.05× (so với -t 10) / 1.30× (so với -t 20)
```

**Tại sao nó work:**

1. **Kiến trúc P-core/E-core và giảm trễ đồng bộ (Barrier Synchronization):** Apple Silicon có 10 physical cores kết hợp giữa Performance cores (P-cores) và Efficiency cores (E-cores). Khi để `-t 10`, luồng xử lý bị phân bổ sang cả các E-cores vốn có tốc độ tính toán và cache L2 thấp hơn, gây ra hiện tượng *straggler effect* (các P-core phải chờ E-core hoàn thành tại các điểm đồng bộ hóa barrier), đồng thời làm tăng chi phí tranh chấp bus bộ nhớ. Khi set `-t 5`, các tác vụ tính toán tập trung trọn vẹn trên các P-cores mạnh nhất.
2. **Bão hòa băng thông bộ nhớ (Memory Bandwidth Saturation) & Tránh Oversubscription:** Do quá trình autoregressive decode bị giới hạn bởi Unified Memory Bandwidth và đã offload GPU (`ngl=99`), chỉ cần 5 threads CPU là đã đủ bão hòa luồng dispatch dữ liệu. Ngược lại, khi nâng lên `-t 20`, hiện tượng oversubscription gây ra chi phí chuyển ngữ cảnh (context switching overhead) và thrashing cache L2 nghiêm trọng, làm sụt giảm thông lượng từ 52.0 xuống 40.0 tok/s (chậm hơn 1.30×).

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** `B2 sweep-quant` (Khảo sát tác động của lượng tử hóa)

**Numbers:**

```
before:  34.6 tok/s (UD-Q6_K_XL - 4.39 GB)
after:   46.9 tok/s (UD-Q4_K_XL - 2.97 GB)
speedup: 1.35×
```

**Điều này nói lên gì mà deck chưa nói:**

Sự gia tăng tốc độ sinh token (decode) từ 34.6 tok/s lên 46.9 tok/s (tăng 1.35×) tỷ lệ nghịch với dung lượng của mô hình (từ 4.39 GB xuống 2.97 GB). Điều này chứng minh bằng thực nghiệm rằng tốc độ decode của LLM ở `batch_size = 1` hoàn toàn bị giới hạn bởi **Băng thông bộ nhớ (Memory Bandwidth Bound)**. Khi ta nén trọng số (weights), số lượng byte cần chuyển từ RAM vào GPU cho mỗi bước tính toán ít hơn, nhờ đó GPU mất ít chu kỳ chờ đợi dữ liệu (memory stalls) và sinh token nhanh hơn. UD-Q4_K_XL chính là điểm "sweet spot" lý tưởng nhất vì nó giữ lại được độ chính xác (quality) xuất sắc trong khi vẫn cho tốc độ dư sức vượt tốc độ đọc của con người, không cần phải hy sinh thêm chất lượng để xuống bản 2-bit.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
