# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=12` `ngl=0` `ctx=2048`
`max_tokens=32` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 10107 | 760 / 1505 | 106.3 / 164.4 | 3973 / 4977 / 4977 | 9.4 |
| UD-Q2_K_XL | 2.24 | 6947 | 889 / 1640 | 91.6 / 134.4 | 3625 / 5238 / 5238 | 10.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.16x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Quan sát

Q2 nhỏ hơn 0.73 GB (24.6%) và decode nhanh hơn 1.16x, dù TTFT P50 tăng từ 760 lên
889 ms. Khi hỏi cùng một câu về throughput/goodput, cả hai bản đều trả lời đúng và
có cấu trúc; Q2 không cho thấy suy giảm rõ trong phép thử nhỏ này. Vì vậy Q2 đáng dùng
cho workload ưu tiên bộ nhớ/tốc độ, nhưng cần bộ đánh giá rộng hơn trước khi dùng thật.
