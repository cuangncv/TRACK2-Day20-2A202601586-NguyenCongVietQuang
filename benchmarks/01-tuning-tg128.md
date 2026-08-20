# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 16.9 | 59% |
| 2 | 23.7 | 82% |
| 4 | 28.8 | 100% |
| 8 | 21.9 | 76% |
| 16 | 2.7 | 10% |

**Best**: `-t 4` at 28.8 tok/s
**Slowest tested**: `-t 16` at 2.7 tok/s (10.49x spread)
**Against the physical-core default** (`-t 4`, 28.8 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

Knee nằm đúng ở -t 4, bằng số core vật lý. Từ 1 đến 4 thread, mỗi thread chạy trên một
core vật lý riêng nên throughput tăng gần tuyến tính. Qua 4 thread, các thread thêm phải
chia sẻ core vật lý qua hyperthreading, tranh nhau cùng đơn vị tính toán và băng thông
bộ nhớ, nên -t 8 chỉ còn 76% đỉnh. Ở -t 16 (gấp đôi số luồng logic), máy phải
oversubscribe nặng, context-switch liên tục, cộng thêm chi phí ảo hoá CPU của WSL2, nên
throughput sập xuống 10% đỉnh thay vì giảm dần đều.
