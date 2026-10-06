# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Lương Quang Huy
**MSSV:** 2A202602698
**Cohort:** K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11
- **CPU:** 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2
- **RAM:** 15.7 GB
- **Accelerator:** Vulkan
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Trên Windows PowerShell, cần thiết lập mã hóa UTF-8 cho console và script để tránh lỗi charmap khi in các ký tự Unicode đồ họa. Sau đó, bản prebuilt llama.cpp b10488 và mô hình Gemma 4 E2B (~5.2 GB) được tải và chạy trực tiếp mượt mà.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:---|---:|---:|---:|---:|---:|---:|
| UD-Q4_K_XL | 2.97 | 64545 | 1309 / 12480 | 190.4 / 204.6 | 13400 / 23155 / 23155 | 5.3 |
| UD-Q2_K_XL | 2.24 | 25599 | 2265 / 19694 | 276.2 / 284.3 | 19671 / 36539 / 36539 | 3.6 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản 2-bit dù nhỏ hơn 0.73 GB nhưng decode chậm hơn 1.47x do CPU phải tốn thêm chu kỳ xử lý dequantize (compute-bound) lớn hơn lượng I/O tiết kiệm được. Chất lượng câu trả lời cũng suy giảm đáng kể. Hoàn toàn không đáng dùng trên máy 16 GB RAM.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|---:|---:|---:|---:|---:|---:|---:|
| 10 | 0.07 | 38000 | 57000 | 57000 | 3.0 | 0.0% |
| 50 | 0.18 | 39000 | 55000 | 55000 | 6.7 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 2.60×
- **P95 tăng:** 0.96×
- **Effective concurrency ở 50 users:** 6.7 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.73 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hòa quanh 50 users khi effective concurrency (6.7) vượt quá 4 slot và có tới 46 request deferred. Khi quá tải, latency tăng thêm chủ yếu là queue time. Để nâng goodput@SLO, tôi sẽ giảm max_tokens hoặc tăng số slot song song kèm KV cache quantization.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|:---|:---|:---|
| N16 Cloud/IaC | Cloud Infrastructure | stub |
| N17 Data pipeline | Ingestion Pipeline | stub |
| N18 Lakehouse | Storage Layer | stub |
| N19 Vector + features | Keyword Retrieval | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 2.6 ms
- llm: 7858.7 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck tập trung 100% ở stage LLM (7.8s so với retrieve 2.6ms), khớp đúng kỳ vọng do CPU inference nặng. Để giảm độ trễ 2x, cần tập trung vào LLM bằng Prefix Caching cho retrieved context và giảm output tokens.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Đổi quantization từ UD-Q2_K_XL sang UD-Q4_K_XL

```
before:  3.6 tok/s (UD-Q2_K_XL)
after:   5.3 tok/s (UD-Q4_K_XL)
speedup: 1.47×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Đổi từ bản 2-bit sang 4-bit giúp tăng tốc decode ấn tượng 1.47x (từ 3.6 tok/s lên 5.3 tok/s), đi ngược lại với suy nghĩ thông thường rằng mô hình càng nhỏ thì chạy càng nhanh.

Cơ chế đằng sau là do trên CPU laptop không có discrete GPU hỗ trợ tensor core, tốc độ decode không chỉ bị giới hạn bởi memory bandwidth mà còn bị giới hạn bởi năng lực tính toán (compute-bound) khi dequantize. Định dạng 2-bit (UD-Q2_K_XL) đòi hỏi các phép unpack bitwise, scaling và zero-point phức tạp, ngốn rất nhiều chu kỳ xử lý của CPU. Trong khi đó, định dạng 4-bit (UD-Q4_K_XL) có block layout tối ưu hơn nhiều cho các lệnh vector AVX2, giải mã trực tiếp với chi phí tính toán thấp hơn hẳn, bù trừ vượt bậc cho lượng dữ liệu 0.73 GB tăng thêm.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều ngạc nhiên nhất là bản lượng tử hóa 2-bit lại chạy chậm hơn bản 4-bit tới gần 1.5 lần trên CPU, chứng minh rằng quantization thấp không phải lúc nào cũng mang lại hiệu năng cao hơn nếu kiến trúc phần cứng bị nghẽn ở compute thay vì bandwidth.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md` đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng AI Assistant để hỗ trợ đọc hướng dẫn, tự động hóa chạy các lệnh đo lường và format bảng biểu báo cáo.
