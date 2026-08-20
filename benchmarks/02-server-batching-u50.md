# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.79 of 4 slots (95%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1168 |

Highest sampled value was **3.79 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Quan sát

Peak busy width là 3.79/4 slots (95%), trong khi effective concurrency là 7.8. Hai số
không mâu thuẫn: gauge đo slot đang decode, còn Little's Law tính cả request chờ. Tôi
tin gauge cho batch width và dùng 46 deferred requests cùng occupancy 7.8/4 làm bằng
chứng queue. Scheduler đã continuous-batch gần kín bốn slot, còn tải dư nằm trong hàng đợi.
