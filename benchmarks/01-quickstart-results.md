# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 10585 | 863 / 1236 | 393.5 / 397.0 | 25631 / 26052 / 26052 | 2.5 |
| UD-Q2_K_XL | 2.24 | 7786 | 792 / 1240 | 302.9 / 308.2 | 19714 / 20465 / 20465 | 3.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.32x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

Bản `UD-Q2_K_XL` (2-bit) nhỏ hơn 0.73 GB (giảm 24.6% dung lượng) và cho tốc độ decode nhanh hơn 1.32× (3.3 tok/s so với 2.5 tok/s, TPOT P50 giảm từ 393.5 ms xuống 302.9 ms). Do TPOT bị nghẽn trực tiếp bởi memory bandwidth khi đọc weights qua RAM mỗi token, quantization thấp hơn giảm trực tiếp số byte cần nạp. Bản 2-bit rất đáng dùng khi cần tối ưu throughput và tiết kiệm RAM trên laptop; bản 4-bit phù hợp hơn khi ưu tiên độ chuẩn xác và ngữ pháp cho prompt phức tạp.
