# Reflection — Day 20 Lab (Personal Report)

**Họ tên:** Tô Thái Dương - 2A202601994

**Cohort:** AI20K-3

**Ngày submit:** 2026-08-20

## 1. Hardware & runtime

- **OS:** Windows 11 Home Single Language (kernel 10.0.26200; Python probe hiển thị Windows 10)
- **CPU:** 12th Gen Intel Core i5-12500H
- **Cores:** 12 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 15.7 GB
- **Accelerator:** Intel Iris Xe Graphics, Vulkan runtime; base measurements dùng CPU-only
- **llama.cpp asset:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL
- **Chạy ở đâu:** laptop cá nhân

**Setup story:** Windows PowerShell 5.1 đọc sai UTF-8 của runner, còn probe ghép cả
thông báo lỗi CIM thành dung lượng RAM. Tôi thêm BOM/UTF-8 output và fallback Win32 API,
sau đó probe đúng i5-12500H, 12/16 cores và 15.7 GB RAM. Vulkan Q2 rất chậm nên base
được đo nhất quán CPU-only.

## 2. Đo lường

Settings: 12 threads, CPU-only (`ngl=0`), context 2048, tối đa 32 output tokens.

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
| ------------ | --------: | --------: | ----------------: | ----------------: | -------------------: | -------------: |
| UD-Q4_K_XL   |      2.97 |     10107 |        760 / 1505 |     106.3 / 164.4 |   3973 / 4977 / 4977 |            9.4 |
| UD-Q2_K_XL   |      2.24 |      6947 |        889 / 1640 |      91.6 / 134.4 |   3625 / 5238 / 5238 |           10.9 |

**Quan sát:** Q2 nhỏ hơn 0.73 GB và decode nhanh hơn 1.16x, dù TTFT P50 cao hơn.
Khi hỏi cùng câu về throughput/goodput, cả hai trả lời đúng và có cấu trúc; Q2 không
giảm chất lượng rõ trong phép thử nhỏ. Q2 đáng dùng ở workload này, nhưng chưa thể suy
rộng thành kết luận chất lượng tổng quát.

## 3. Serving under load

Server dùng `--parallel 4`, 6 threads và CPU-only.

| Users |  RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
| ----: | ---: | -------: | -------: | -------: | ---------------: | -------: |
|    10 | 0.15 |    42000 |    53000 |    53000 |              5.4 |     0.0% |
|    50 | 0.22 |    36000 |    54000 |    54000 |              7.8 |     0.0% |

- **Offered load tăng 5x, throughput thực tăng:** 1.52x
- **P95 tăng:** 1.02x
- **Effective concurrency ở 50 users:** 7.8 so với 4 slots
- **Peak `n_busy_slots_per_decode`:** 3.79/4 slots; peak deferred requests: 46

**Saturation reading:** Server bão hòa ở hoặc dưới 10 users vì effective concurrency
5.4 đã vượt 4 slots; ở 50 users busy width đạt 95% và có 46 request deferred. Offered
load tăng 5x nhưng RPS chỉ tăng 1.52x. P95 bị lệch bởi chỉ tính request hoàn thành.
Tôi sẽ thử `--parallel 8`, đo goodput@SLO và rollback nếu contention làm TPOT xấu hơn.

## 4. Integration

| Day                   | Piece                      | Real hay stub? |
| --------------------- | -------------------------- | -------------- |
| N16 Cloud/IaC         | localhost                  | stub           |
| N17 Data pipeline     | in-memory list             | stub           |
| N18 Lakehouse         | toy dictionary             | stub           |
| N19 Vector + features | TOY_DOCS + keyword overlap | stub           |
| N20 Serving           | llama-server               | real           |

Mean của ba query:

- **embed:** 0.0 ms
- **retrieve:** 0.1 ms
- **llm:** 6046.2 ms
- **total:** 6046.3 ms
- **stage lớn nhất:** LLM, xấp xỉ 100% total

**Reflection:** Bottleneck là LLM, đúng kỳ vọng vì embedding không chạy và retrieval
toy gần như miễn phí. Muốn giảm pipeline latency 2x, tôi tối ưu decode trước bằng Q2,
6 threads và output budget phù hợp; tối ưu 0.1 ms retrieval không tạo khác biệt đáng kể.

## 5. The single change that mattered most

**Change:** giảm decode threads từ mặc định 12 xuống 6.

```text
before:  9.3 tok/s tại 12 threads
after:   9.6 tok/s tại 6 threads
speedup: 1.03x
```

Decode phải stream trọng số qua memory hierarchy cho từng token nên nhanh chóng bị
giới hạn bởi memory bandwidth, không phải thiếu FLOPs. Sau knee, thêm threads làm các
worker tranh băng thông và cache, đồng thời tăng synchronization/scheduling overhead.
CPU i5-12500H là kiến trúc hybrid, nên đếm đủ 12 physical cores không bảo đảm 12 worker
có hiệu quả ngang nhau. Vì vậy 6 threads đạt 9.6 tok/s, 12 chỉ 9.3, 16 còn 8.5 và
oversubscribe 32 threads giảm mạnh xuống 6.0 tok/s.

Speedup 1.03x không lớn, nhưng curve cho thấy cơ chế rõ: điểm tối ưu là knee trước khi
tranh chấp tài nguyên lấn át lợi ích song song. Nó cũng tránh chọn 32 threads, cấu hình
chậm hơn 37.5% so với optimum.

## 6. Bonus (optional)

Không thực hiện; ưu tiên hoàn tất và kiểm chứng base track.

## 7. Điều làm tôi ngạc nhiên nhất

Quantization 2-bit nhanh hơn trên CPU nhưng cực chậm khi offload lên Intel Iris Xe
Vulkan trong lượt thử đầu; “có GPU offload” không tự động đồng nghĩa với nhanh hơn.

## 8. Self-check

- Hardware/model manifests và toàn bộ benchmark artifacts đã tạo.
- Các section bắt buộc trong reports đã được điền.
- Đã thêm đủ 5 screenshots thật, xác nhận cohort và chạy `lab.ps1 verify` thành công.
