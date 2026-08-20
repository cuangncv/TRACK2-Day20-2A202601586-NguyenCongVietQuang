# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 8363.8 | 8363.9 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5664.4 | 5664.5 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 4673.7 | 4673.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **6234.0** · total **6234.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it specifically accounts for **SLOs (Service Level Objects)** and **TPOT (Throughput at Saturation)**.

Here is the breakdown of why this makes it superior:

1.  **Accurate Measurement of Actual Load**: Raw throughput measures the total requests processed per second (TPOT). However, it ignores the specific constr

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation and memory bandwidth bottlenecks** in GPU memory.

Specifically, it addresses two major issues:
1.  **Internal Fragmentation**: By storing KV cache in non-contiguous pages, it removes the wasted space that would otherwise be allocated to the GPU's internal memory.
2.  **Compute-Bound vs. Memory-Bound Splits**: It enables the engine to s

**When does splitting prefill and decode help?**

> When prefill and decode are split, the system avoids computing the full token prefix for the entire batch. Specifically, this optimization is triggered when **continuous batching** is enabled.

In this scenario, requests join and leave the running batch each time the decode step occurs, rather than waiting for a complete batch to be processed. By splitting the prefill (which is compute-bound) and 


## Which N16-N19 pieces are real

N16 (Cloud/IaC): stub, không dùng trong pipeline này.
N17 (Data pipeline): stub, không dùng trong pipeline này.
N18 (Lakehouse): stub, không dùng trong pipeline này.
N19 (Vector + features): stub, dùng keyword overlap fallback thay vì embedding + vector
search thật.
N20 (Serving): real, gọi thẳng `llama-server`.

Dominant stage (llm, 100% total) đúng như kỳ vọng: retrieve chỉ so khớp từ khoá nên gần
như free (0.0-0.1ms), còn máy không có GPU offload nên decode hàng trăm token mất vài
giây. Nếu phải giảm latency pipeline 2 lần, sẽ tấn công vào stage llm trước — giảm
`max_tokens` hoặc bật GPU offload (`-ngl`) cho GPU MX330 sẵn có, vì đó là nơi gần như
100% thời gian đang mất.
