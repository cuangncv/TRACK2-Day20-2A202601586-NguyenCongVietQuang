# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2106 | 284 / 364 | 34.7 / 41.2 | 2455 / 2962 / 2962 | 28.8 |
| UD-Q2_K_XL | 0.39 | 2040 | 614 / 1503 | 48.5 / 66.8 | 3693 / 4706 / 4706 | 20.6 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.40x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Quant nhỏ hơn (UD-Q2_K_XL) không đáng dùng trên máy này. Nó chậm hơn Q4_K_M 1.40 lần
(TTFT P50 614ms so với 284ms, TPOT P50 48.5ms so với 34.7ms) dù chỉ nhẹ hơn 0.11GB. Máy
chỉ có 4 core vật lý, không GPU offload, nên bị giới hạn bởi compute chứ không phải
memory bandwidth — chi phí dequantize thêm của Q2 lớn hơn phần bytes tiết kiệm được.
Chưa test A/B trực tiếp chất lượng câu trả lời trong lần chạy này, chỉ so trên số đo
được ở trên.
