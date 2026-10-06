# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 2.7 | 98% |
| 4 | 2.7 | 99% |
| 8 | 2.7 | 97% |
| 16 | 2.8 | 100% |
| 32 | 2.7 | 97% |

**Best**: `-t 16` at 2.8 tok/s
**Slowest tested**: `-t 8` at 2.7 tok/s (1.03x spread)
**Against the physical-core default** (`-t 8`, 2.7 tok/s): 1.03x

Use this in your run:

```bash
LAB_N_THREADS=16 make bench
```

## Your explanation

Đường cong decode (`tg128`) gần như phẳng hoàn toàn giữa các mức thread (2.7 đến 2.8 tok/s, độ biến thiên chỉ 1.03×), đạt đỉnh nhẹ ở `-t 16` (ứng với 16 logical cores) và giảm nhẹ ở `-t 32`. Hiện tượng đường cong đi ngang thay vì tăng mạnh theo core count bắt nguồn từ hai cơ chế: (1) Quá trình decode bị nghẽn nghiêm ngặt bởi memory bandwidth (Memory-bound) — việc đọc toàn bộ weights qua bus RAM mỗi token đã làm nghẽn kênh nhớ ngay từ số luồng thấp; (2) Hệ thống sử dụng GPU offload Vulkan (`ngl=99`), do đó nhân đồ họa tích hợp Radeon đảm nhận tính toán ma trận chính, giảm sự phụ thuộc vào số thread CPU. Khi oversubscribe lên 32 threads, hiện tượng tranh chấp cache và context-switching bắt đầu làm giảm nhẹ hiệu năng.
