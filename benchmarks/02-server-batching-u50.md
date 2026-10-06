# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.51 of 4 slots (88%) |
| `requests_processing` | 0 |
| `requests_deferred` | 0 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 448 |

Highest sampled value was **3.51 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` stayed at zero: every request found a free slot on arrival.

## Your observation

- **Peak Batch Width:** Đạt đỉnh **3.51 trên tổng số 4 slots** vật lý (`--parallel 4`), tương đương hệ số tận dụng slot đạt **88%**. Điều này chứng minh thuật toán Continuous Batching (cellular/iteration-level scheduling) của `llama-server` đã hoạt động hiệu quả: thay vì tuần tự hóa từng request hoặc gom batch tĩnh, scheduler đã gộp các sequence độc lập vào chung các bước forward pass decode, giúp tăng token throughput tổng thể.
- **So sánh với Effective Concurrency trong `02-server-results.md`:**
  - Trong `02-server-results.md`, Effective Concurrency (tính theo Định luật Little: $L = \lambda \cdot W$) ghi nhận là **5.4** tại 50 users (vượt qua 4 slots, occupancy ratio 1.35).
  - Có sự chênh lệch giữa hai số đo: **3.51** (internal engine gauge) so với **5.4** (client-side Little's Law).
- **Độ tin cậy và giải thích nguyên nhân khác biệt:**
  - **`n_busy_slots_per_decode` (3.51)** là metric đo trực tiếp từ bên trong kernel của engine tính toán `llama.cpp` cho mỗi bước decode step. Vì server chỉ khởi tạo 4 compute slots (`--parallel 4`), số lượng slot tham gia tính toán forward pass đồng thời không thể vượt quá 4.0. Giá trị 3.51 phản ánh chính xác hiệu suất batching thực tế của compute engine.
  - **Little's Law concurrency (5.4)** được đo từ góc nhìn client bên ngoài (Locust HTTP layer). Giá trị này bao gồm toàn bộ thời gian trong hệ thống (System Residency Time = Queueing Time + Compute Time + HTTP overhead). Khi số lượng request gửi tới đồng thời vượt quá 4 slots khả dụng, các request phải xếp hàng chờ trong hàng đợi kết nối HTTP của server trước khi chiếm được slot. Do đó, $L > 4$ phản ánh tình trạng quá tải (oversubscribed / queue buildup).
  - **Kết luận:** Ta tin cậy `n_busy_slots_per_decode` (3.51/4) để đánh giá khả năng bão hòa năng lực tính toán và batching của engine, trong khi Little's Law (5.4) là bằng chứng xác thực cho thấy server đã vượt ngưỡng phục vụ và bắt đầu phát sinh hàng đợi.
