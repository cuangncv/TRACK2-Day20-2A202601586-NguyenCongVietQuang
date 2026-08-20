# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 28 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.80 of 4 slots (95%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 3677 |

Highest sampled value was **3.80 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width là 3.80 của 4 slot (95%), trong khi effective concurrency trong
`02-server-results.md` ở 50 user là 15.0. Hai số này không mâu thuẫn, vì chúng đo hai
thứ khác nhau: `n_busy_slots_per_decode` là gauge thật của server, đo đúng bao nhiêu
slot decode đang bận tại một thời điểm, nên không thể vượt quá 4 (số `--parallel`).
Effective concurrency tính theo Little's Law (RPS x latency trung bình) đếm luôn cả
request đang xếp hàng chờ (`requests_deferred` = 46), không chỉ request đang được decode,
nên số này lớn hơn nhiều so với số slot thật. Tin `n_busy_slots_per_decode` khi muốn biết
động cơ decode có bận hết công suất không (câu trả lời: gần như có, 95%). Tin effective
concurrency khi muốn biết tổng áp lực nhu cầu đang dồn vào server, kể cả phần đang xếp
hàng — đó chính là phần queue time cộng thêm vào P95.
