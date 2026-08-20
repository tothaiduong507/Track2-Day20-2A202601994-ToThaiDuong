# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 8 | 0.15 | 42000 | 53000 | 53000 | 5.4 | 0.0% |
| 50 | 12 | 0.22 | 36000 | 54000 | 54000 | 7.8 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.52x** (30% of linear) |
| P95 latency | **1.02x** |
| Effective concurrency at 50 users | 7.8 vs `--parallel 4` slots (occupancy/slot ratio 1.96) |

**Saturated.** Throughput delivered only 1.52x for 5x the offered load, and effective concurrency (7.8) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (1.02x vs 1.52x), so this server still has headroom at 50 users.

> **Small sample.** Only 8 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Nhận định

Server đã bão hòa ở hoặc dưới 10 users: effective concurrency 5.4 đã vượt 4 slots; ở
50 users nó lên 7.8, busy width đạt 3.79/4 và có 46 request deferred. Offered load tăng
5x nhưng throughput chỉ tăng 1.52x. P95 gần như phẳng do thống kê chỉ chứa request đã
hoàn thành, trong khi request còn queue bị cắt khi hết 60 giây. Tôi sẽ thử `--parallel 8`
trước và đo lại goodput@SLO: bằng chứng hiện tại cho thấy giới hạn trực tiếp là bốn slot,
nhưng sẽ rollback nếu contention làm TPOT/P95 xấu hơn.
