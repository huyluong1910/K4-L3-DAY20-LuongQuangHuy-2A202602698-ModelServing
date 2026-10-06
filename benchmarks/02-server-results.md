# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` Â· llama.cpp `b10488` Â·
`--parallel 4` Â· `ctx=2048` Â· `threads=4` Â·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 4 | 0.07 | 38000 | 57000 | 57000 | 3.0 | 0.0% |
| 50 | 10 | 0.18 | 39000 | 55000 | 55000 | 6.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **2.60x** (52% of linear) |
| P95 latency | **0.96x** |
| Effective concurrency at 50 users | 6.7 vs `--parallel 4` slots (occupancy/slot ratio 1.66) |

**At capacity, still scaling.** All 4 decode slots are busy (effective concurrency 6.7) but throughput still rose 2.60x. You are at the knee -- the next increment of load is where P95 starts to run away.

P95 grew no faster than throughput (0.96x vs 2.60x), so this server still has headroom at 50 users.

> **Small sample.** Only 4 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

Server chạm điểm bão hòa (knee of saturation) quanh mức 50 users khi Effective Concurrency theo Little's Law đạt 6.7, vượt qua dung lượng 4 slots của --parallel 4 (tỉ lệ occupancy 1.66). Bằng chứng thuyết phục nhất là số lượng 
equests_deferred tăng vọt lên 46 trong khi throughput thực tế chỉ tăng 2.60x (dù tải tăng 5x, đạt 52% mức tuyến tính).
Độ trễ P50 duy trì quanh 38-39s nhưng khi vượt quá 4 slot, các request phải chờ đợi trong queue.
Để nâng cao goodput@SLO, knob đầu tiên tôi sẽ điều chỉnh là:
1. **Giảm max_tokens hoặc áp dụng dynamic prompt batching**: Rút ngắn thời gian chiếm dụng slot của từng request, giải phóng slot nhanh hơn cho hàng đợi.
2. Hoặc nếu máy đủ VRAM/RAM, tăng --parallel từ 4 lên 6-8 slots kèm theo KV cache quant (k-quant/v-quant) để phục vụ đồng thời nhiều luồng hơn mà không gây tràn bộ nhớ.
