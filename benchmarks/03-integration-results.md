# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 14837.0 | 14837.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 11992.8 | 11992.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 12373.3 | 12373.4 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **13067.7** · total **13067.8**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **Danh sách trạng thái các thành phần N16 - N19:**
  - **N16 Cloud/IaC:** **Stub** (chạy cục bộ trên laptop cá nhân với môi trường Windows và AMD Ryzen APU, không dựng hạ tầng cloud).
  - **N17 Data pipeline:** **Stub** (sử dụng danh sách dữ liệu mẫu `TOY_DOCS` nạp sẵn trong script).
  - **N18 Lakehouse:** **Stub** (in-memory dictionary storage thay vì Apache Iceberg/Delta Lake).
  - **N19 Vector + features:** **Stub** (sử dụng thuật toán so khớp từ khóa keyword overlap trên bộ docs mẫu).
  - **N20 Serving:** **Real** (`llama-server` b10488 phục vụ mô hình Gemma 4 E2B thật qua endpoint HTTP OpenAI `/v1/chat/completions`).

- **Đánh giá về Dominant Stage:**
  - Stage chiếm tỷ trọng lớn nhất là **`llm` với 13,067.7 ms (chiếm 100.0% tổng thời gian pipeline)**, trong khi `embed` và `retrieve` chỉ tiêu tốn 0.0 - 0.1 ms.
  - Kết quả này **hoàn toàn đúng với kỳ vọng ban đầu**: Giai đoạn retrieval chỉ thực hiện so khớp từ khóa nhẹ trên RAM với độ phức tạp $O(N)$, trong khi `llm` phải nạp 2.97 GB trọng số mô hình qua bus bộ nhớ RAM tuần tự cho từng token được giải mã (autoregressive decode ~23-30 tokens/câu hỏi), tạo nên nút thắt cổ chai về memory bandwidth.

- **Kế hoạch giảm độ trễ pipeline xuống 2× (Halve the latency):**
  - **Giai đoạn tấn công:** Bắt buộc phải tập trung toàn lực vào **`llm` stage** (theo Định luật Amdahl, tối ưu hóa retrieval không thể mang lại cải thiện có ý nghĩa vì retrieval chỉ chiếm <0.01% tổng thời gian).
  - **Giải pháp cụ thể:**
    1. **Giới hạn số token sinh ra (`max_tokens` / Early stopping):** Yêu cầu mô hình trả lời cô đọng trong 1 câu chính xác (như output thực tế chỉ cần ~20 tokens). Vì mỗi token tốn gần 400 ms decode, việc giảm 15 token sinh thừa sẽ cắt giảm trực tiếp 6,000 ms.
    2. **Sử dụng trọng số 2-bit (`UD-Q2_K_XL`):** Giảm TPOT từ 393.5 ms xuống 302.9 ms, lập tức mang lại tốc độ decode nhanh hơn 1.32×.
    3. **Tận dụng KV Cache Prefix Caching:** Giữ lại KV cache của `SYSTEM_PROMPT` và context tài liệu dùng chung giữa các query để triệt tiêu thời gian prefill (~1,600 - 1,850 ms).
