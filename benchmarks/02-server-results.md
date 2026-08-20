# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 31 | 0.53 | 16000 | 22000 | 23000 | 8.3 | 0.0% |
| 50 | 31 | 0.54 | 20000 | 56000 | 57000 | 15.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.01x** (20% of linear) |
| P95 latency | **2.55x** |
| Effective concurrency at 50 users | 15.0 vs `--parallel 4` slots (occupancy/slot ratio 3.74) |

**Saturated.** Throughput delivered only 1.01x for 5x the offered load, and effective concurrency (15.0) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.01x while P95 moved 2.55x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Server bão hoà ở khoảng 10 user trở xuống, không phải 50. Con số thuyết phục nhất:
throughput gần như không đổi giữa 10 và 50 user (0.53 → 0.54 RPS, 1.01x) dù offered load
tăng 5 lần, trong khi P95 tăng 2.55x và effective concurrency (15.0) vượt xa 4 slot của
`--parallel`. `make metrics` đo peak `n_busy_slots_per_decode` = 3.80/4 khi chạy 50 user
— server luôn bận gần kịch trần. Phần latency tăng thêm là queue time chứ không phải
compute time, vì tốc độ decode mỗi token không đổi giữa hai lần chạy. Nếu cần nâng
goodput ở một SLO P95, việc đầu tiên nên đổi là tăng `--parallel` (số slot song song),
vì bottleneck nằm ở số request được phục vụ đồng thời chứ không phải tốc độ decode.
