# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Trung Kiên
**MSSV:** 2A202602764
**Cohort:** K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (AMD64)
- **CPU:** AMD Ryzen 7 5700U with Radeon Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 14.8 GB
- **Accelerator:** Vulkan
- **llama.cpp asset đã tải:** llama-b10488-bin-win-vulkan-x64.zip
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Môi trường Windows chạy bằng PowerShell qua `lab.ps1`. Ban đầu script gặp lỗi encoding console cp1252 khi in ký tự Unicode nên đã set `$env:PYTHONUTF8=1`. Binary runtime llama.cpp b10488 Vulkan và model Gemma 4 E2B được tải trực tiếp thành công.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 10585 | 863 / 1236 | 393.5 / 397.0 | 25631 / 26052 / 26052 | 2.5 |
| UD-Q2_K_XL | 2.24 | 7786 | 792 / 1240 | 302.9 / 308.2 | 19714 / 20465 / 20465 | 3.3 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

2-bit nhanh hơn 1.32× và nhẹ hơn 0.73 GB. Rất đáng dùng trên laptop vì TPOT bị nghẽn bởi memory bandwidth. Thử nghiệm cho thấy bản 2-bit trả lời mạch lạc cho câu hỏi thông thường, nhưng bản 4-bit suy luận logic và ngữ pháp chính xác hơn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.17 | 45000 | 46000 | 46000 | 6.1 | 0.0% |
| 50 | 0.16 | 39000 | 50000 | 50000 | 5.4 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.91×
- **P95 tăng:** 1.09×
- **Effective concurrency ở 50 users:** 5.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.51 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hòa ngay từ ≤ 10 users; tăng 5× tải nhưng throughput chỉ đạt 0.91× (18% tuyến tính). Concurrency 5.4 > 4 slots và peak busy slots 3.51/4 (88%) chứng minh slots đã nghẽn, độ trễ P95 tăng từ 46s lên 50s hoàn toàn do queue time. Để nâng goodput@SLO, ưu tiên đổi quantization sang UD-Q2_K_XL trước nhằm tăng tốc decode 1.32×, giải phóng slot nhanh hơn mà không nghẽn memory bandwidth như việc tăng threads.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local Windows workstation | stub |
| N17 Data pipeline | In-memory `TOY_DOCS` | stub |
| N18 Lakehouse | Python dict storage | stub |
| N19 Vector + features | Keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 13067.7 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck nằm 100% ở stage llm (13,067 ms) do autoregressive decode bị nghẽn băng thông bộ nhớ khi đọc weights mỗi token, hoàn toàn khớp kỳ vọng. Để giảm latency 2×, tôi sẽ tấn công trực tiếp vào stage llm: chuyển sang UD-Q2_K_XL (nhanh hơn 1.32×), giới hạn output length ngắn gọn và bật prefix caching.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Chuyển đổi quantization từ `UD-Q4_K_XL` (4-bit) sang `UD-Q2_K_XL` (2-bit)

```
before:  2.5 tok/s (TPOT 393.5 ms)
after:   3.3 tok/s (TPOT 302.9 ms)
speedup: 1.32×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Trong pha autoregressive decode, mô hình bị ràng buộc tuyệt đối bởi băng thông bộ nhớ (Memory-Bandwidth Bound). Với mỗi token sinh ra, toàn bộ trọng số của mô hình phải được tải tuần tự từ RAM chính vào cache/ALU với tỷ lệ tính toán cực thấp (Arithmetic Intensity ~ 1 FLOP/Byte). Khi chuyển từ bản 4-bit (2.97 GB) sang 2-bit (2.24 GB), khối lượng dữ liệu di chuyển qua bus nhớ giảm trực tiếp 24.6%, giúp TPOT P50 giảm từ 393.5 ms xuống 302.9 ms, đạt speedup 1.32× mà không cần can thiệp phần cứng.

Điều này cũng giải thích kết quả từ `make tune` (`01-tuning-tg128.md`): việc tăng số luồng CPU từ `-t 1` lên `-t 8` hay `-t 16` chỉ tăng nhẹ từ 2.7 lên 2.8 tok/s (1.03×), bởi vì băng thông kênh nhớ của RAM và GPU Vulkan (`ngl=99`) đã bão hòa ngay từ số thread thấp; thêm luồng tính toán không thể giải quyết bài toán thiếu băng thông đọc weights. Do đó, giảm kích thước weight (quantization) là sự thay đổi mang lại tác động lớn nhất và hiệu quả nhất trên hệ thống.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều làm tôi ngạc nhiên nhất là việc tăng số luồng CPU từ 1 lên 16 hay 32 luồng gần như không hề cải thiện tốc độ sinh token (vẫn giữ nguyên ~2.7 - 2.8 tok/s ở Step 1.2), chứng minh rõ ràng trên thực nghiệm rằng bài toán autoregressive decode LLM bị nghẽn tuyệt đối bởi băng thông bộ nhớ RAM DDR4 chứ không phải năng lực tính toán của CPU.

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
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng Antigravity IDE (Gemini) để hỗ trợ cấu hình script chạy benchmark trên môi trường Windows PowerShell, tự động hóa load testing và định dạng báo cáo.
