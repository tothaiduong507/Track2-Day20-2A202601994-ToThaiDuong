# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **12 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 2.3 | 24% |
| 6 | 9.6 | 100% |
| 12 | 9.3 | 97% |
| 16 | 8.5 | 89% |
| 32 | 6.0 | 63% |

**Best**: `-t 6` at 9.6 tok/s
**Slowest tested**: `-t 1` at 2.3 tok/s (4.10x spread)
**Against the physical-core default** (`-t 12`, 9.3 tok/s): 1.03x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Giải thích

Knee nằm ở 6 threads (9.6 tok/s), không phải 12 physical cores. Decode phải đọc trọng
số lặp lại nên bị giới hạn bởi memory bandwidth; thêm threads sau khi băng thông/cache
đã bão hòa chỉ tăng cạnh tranh, đồng bộ và scheduling overhead. CPU i5-12500H còn là
kiến trúc hybrid, nên số physical cores không đồng nghĩa với số worker hiệu quả như nhau.
Điều này phù hợp với việc 16 threads còn 8.5 tok/s và oversubscribe 32 threads chỉ còn
6.0 tok/s. So với mặc định 12 threads, đổi sang 6 threads tạo speedup 1.03x.
