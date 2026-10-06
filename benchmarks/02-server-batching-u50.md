# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` Â· `--parallel 4` Â· 13 samples over
60s at 2.0s intervals Â· raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.73 of 4 slots (93%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a â€” not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1033 |

Highest sampled value was **3.73 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak 
_busy_slots_per_decode đạt tới 3.73 trên tổng số 4 slots (93.3% dung lượng batch). Số lượng 
equests_processing chạm trần tối đa 4 slots và 
equests_deferred lên tới 46 requests phải nằm chờ trong hàng đợi.
Con số này chứng minh thuyết phục rằng cơ chế Continuous Batching của llama-server đang hoạt động thực sự: scheduler liên tục đóng gói các request đến đồng thời vào cùng một bước decode để tận dụng tối đa bandwidth. Khi hệ thống nhận 50 users (vượt quá 4 slots phục vụ), các request dư thừa bị xếp vào hàng chờ, dẫn đến phần lớn độ trễ P95 tăng thêm là thời gian chờ (queue time) chứ không phải do mô hình tính toán chậm hơn (compute time).
