# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 8 | 0.17 | 45000 | 46000 | 46000 | 6.1 | 0.0% |
| 50 | 8 | 0.16 | 39000 | 50000 | 50000 | 5.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.91x** (18% of linear) |
| P95 latency | **1.09x** |
| Effective concurrency at 50 users | 5.4 vs `--parallel 4` slots (occupancy/slot ratio 1.35) |

**Saturated.** Throughput delivered only 0.91x for 5x the offered load, and effective concurrency (5.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.91x while P95 moved 1.09x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 8 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

- **Điểm bão hòa (Saturation Point) và Bằng chứng định lượng:**
  - Server bị bão hòa ngay từ ngưỡng **$\le 10$ người dùng đồng thời**, và bão hòa hoàn toàn ở 50 người dùng.
  - **Con số thuyết phục nhất:** Tỷ lệ tăng trưởng thông lượng thực tế chỉ đạt **0.91×** (từ 0.17 req/s xuống 0.16 req/s) khi tải tăng gấp **5×** (từ 10 lên 50 users), tương đương hiệu quả mở rộng thông lượng chỉ đạt **18% so với tuyến tính**.
  - Đồng thời, độ trễ P95 tăng từ 46,000 ms lên 50,000 ms, và Little's Law Effective Concurrency đạt **5.4 - 6.1**, vượt qua số slots phần cứng cấp phát (`--parallel 4`, occupancy/slot ratio 1.35). Khi các slots tính toán bị chiếm trọn bởi các request đang decode (ở tốc độ ~2.5 - 3.3 tok/s), bất kỳ request mới nào cũng phải chịu độ trễ hàng đợi (queue time) đáng kể.

- **Đánh giá Goodput theo SLO:**
  - Giả sử hệ thống đặt mục tiêu **SLO P95 latency $\le 30$ giây**.
  - Tại 10 users và 50 users, độ trễ P95 đều ở mức 46 - 50 giây, đồng nghĩa **Goodput tại ngưỡng SLO 30s tụt về gần 0%** (các request hoàn thành nhưng đã vi phạm cam kết chất lượng dịch vụ). Quá điểm bão hòa, việc nhồi thêm tải không sinh thêm giá trị mà chỉ làm phình to P95 latency do tích lũy hàng đợi.

- **Knob cần điều chỉnh đầu tiên để nâng cao Goodput:**
  - **Knob ưu tiên số 1: Giảm kích thước Quantization (chuyển sang `UD-Q2_K_XL` hoặc quant hóa KV cache sang q8_0/q4_0).**
  - **Lý do chọn knob này mà không phải các knob khác:**
    - *Vì sao không tăng thread (`-t`)?* Bài kiểm tra sweep threads ở Bước 1.2 đã chứng minh tốc độ decode hoàn toàn đi ngang ở mọi mức thread (2.7 tok/s từ 1 đến 32 threads) do kiến trúc APU chia sẻ băng thông RAM DDR4 (memory-bandwidth bound). Tăng thread chỉ làm tăng overhead chuyển đổi ngữ cảnh CPU.
    - *Vì sao không tăng slots (`--parallel 8`)?* Tăng slots trên một hệ thống nghẽn băng thông bộ nhớ sẽ chia nhỏ dung lượng KV cache và băng thông DRAM khả dụng cho từng slot, khiến tốc độ sinh token của mỗi sequence càng chậm hơn, kéo dài thời gian chiếm giữ slot và làm tăng P95.
    - *Hiệu quả của việc giảm Quantization:* Ở Bước 1.1, `UD-Q2_K_XL` đã chứng minh tăng tốc decode **1.32×** (3.3 tok/s so với 2.5 tok/s của Q4) và giảm TPOT P50 từ 393.5 ms xuống 302.9 ms nhờ giảm dung lượng weights phải nạp qua bus bộ nhớ trong mỗi bước decode. Tốc độ decode nhanh hơn giúp mỗi slot giải phóng sớm hơn, trực tiếp triệt tiêu hàng đợi, kéo P95 latency xuống dưới ngưỡng SLO 30s và khôi phục Goodput cho hệ thống.
