# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Công Việt Quang
**Cohort:** 2A202601586_A20-K4
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Ubuntu (WSL2 trên Windows 11), kernel 6.6.87.2-microsoft-standard-WSL2
- **CPU:** Intel Core i5-1035G1 @ 1.00GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2, AVX-512
- **RAM:** 5.8 GB
- **Accelerator:** CPU only (ngl=0). Có NVIDIA GeForce MX330 2GB nhưng base track không dùng GPU offload
- **llama.cpp asset đã tải:** prebuilt release build b10488, Linux x86_64
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (WSL2 trên Windows)

**Setup story** (≤ 80 chữ): Máy vật lý có 8GB RAM nhưng WSL2 chỉ cấp 3.7GB mặc định.
File `.wslconfig` đã có `memory=6.5GB` nhưng WSL không parse được số thập phân nên bỏ
qua, rơi về mặc định 50% RAM host. Sửa thành `memory=6GB` (số nguyên) rồi `wsl --shutdown`
lại thì lên đúng 5.8GB, đủ ngưỡng chạy Qwen3.5 0.8B local, không cần chuyển sang cloud.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2106 | 284 / 364 | 34.7 / 41.2 | 2455 / 2962 / 2962 | 28.8 |
| UD-Q2_K_XL | 0.39 | 2040 | 614 / 1503 | 48.5 / 66.8 | 3693 / 4706 / 4706 | 20.6 |

**Quan sát** (≤ 60 chữ): quant nhỏ hơn (Q2) chậm hơn quant lớn (Q4) 1.40 lần dù nhẹ
hơn 0.11GB, vì máy này chỉ 4 core vật lý, không GPU offload, nên bị giới hạn bởi
compute chứ không phải memory bandwidth — phần dequantize thêm của Q2 tốn hơn phần
bytes tiết kiệm được. Trên máy này, dùng Q4_K_M hợp lý hơn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.53 | 16000 | 22000 | 23000 | 8.3 | 0.0% |
| 50 | 0.54 | 20000 | 56000 | 57000 | 15.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.01×
- **P95 tăng:** 2.55×
- **Effective concurrency ở 50 users:** 15.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.80 / 4 slots

**Saturation reading** (≤ 80 chữ): server bão hoà ở khoảng 10 user trở xuống, không
phải 50. Bằng chứng: throughput gần như không đổi (0.53 → 0.54 RPS) dù offered load
tăng 5 lần, trong khi P95 tăng 2.55 lần và effective concurrency (15.0) vượt xa 4 slot.
`make metrics` đo peak busy slots 3.80/4 — server luôn bận gần kịch trần. Phần latency
tăng thêm là queue time, không phải compute time, vì TPOT mỗi token không đổi. Muốn
nâng goodput@SLO, việc đầu tiên nên đổi là tăng `--parallel` (số slot), vì bottleneck
nằm ở số request xử lý đồng thời chứ không phải tốc độ decode từng token.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không dùng trong pipeline này | stub |
| N17 Data pipeline | không dùng trong pipeline này | stub |
| N18 Lakehouse | không dùng trong pipeline này | stub |
| N19 Vector + features | keyword overlap fallback, không phải embedding + vector search thật | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.03 ms
- llm: 6234.0 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck nằm hoàn toàn ở llm decode, đúng như kỳ vọng vì
retrieve chỉ so khớp từ khoá (gần như free) còn máy không có GPU offload nên sinh token
chậm. Nếu phải giảm latency 2×, sẽ tấn công vào stage llm trước — giảm `max_tokens`
hoặc bật GPU offload (`-ngl`) cho card MX330 sẵn có, thay vì tối ưu retrieve.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ -t từ 16 xuống 4 (physical core count) trong `make tune`

```
before:  2.7 tok/s   (-t 16)
after:   28.8 tok/s  (-t 4)
speedup: 10.5×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

CPU máy này có 4 core vật lý, 8 luồng logic qua hyperthreading. Từ -t 1 đến -t 4,
throughput tăng gần tuyến tính (16.9 → 23.7 → 28.8 tok/s) vì mỗi thread được một core
vật lý riêng, không tranh nhau. Qua khỏi 4 thread, các thread thêm phải dùng chung core
vật lý qua hyperthreading — chúng tranh nhau cùng một đơn vị tính toán, cache L1/L2 và
băng thông bộ nhớ, nên không thêm compute thật mà chỉ thêm tranh chấp. Ở -t 8, throughput
tụt còn 76% đỉnh. Ở -t 16 (gấp đôi số luồng logic), hệ điều hành phải oversubscribe nặng,
context-switch liên tục giữa quá nhiều thread trên chỉ 8 lõi logic, cộng thêm việc máy
đang chạy trong WSL2 (lịch CPU bị ảo hoá thêm một lớp qua host Windows) khiến chi phí
scheduling càng lớn — đó là lý do throughput sập hẳn xuống 10% đỉnh thay vì chỉ giảm dần
như một đường cong "diminishing returns" thông thường.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
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

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
