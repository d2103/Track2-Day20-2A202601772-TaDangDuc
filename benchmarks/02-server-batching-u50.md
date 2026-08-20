# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 13 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.98 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 18530 |

Highest sampled value was **3.98 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Nhận xét của tôi

**Peak batch width = 3.98 / 4 slots (100%).** Continuous batching hoạt động đúng như mô
tả: scheduler gói gần như tối đa 4 request vào chung mỗi decode step, liên tục suốt cả
cửa sổ đo. Giá trị này ổn định ở 3.96-3.98 trong toàn bộ 13 mẫu, không phải một đỉnh
nhọn thoáng qua.

**Nó có khớp với effective concurrency 40.7 trong `02-server-results.md` không? Không,
và hai con số đó không mâu thuẫn: chúng đo hai thứ khác nhau.**

- `n_busy_slots_per_decode` = 3.98 đo **phía server**: có bao nhiêu slot đang thực sự
  decode. Trần cứng của nó là `--parallel 4`, nên nó **không bao giờ** vượt 4 dù load
  lớn đến đâu.
- Effective concurrency = 40.7 (Little's Law, đo **phía client**) đếm số request đang
  nằm trong **cả hệ thống**: cả đang được phục vụ lẫn đang xếp hàng.

Chênh lệch **40.7 trừ 4, tức khoảng 37 request đang chờ**, và server tự xác nhận điều
đó: `requests_deferred` giữ ở **40-45** trong mọi mẫu. Đây chính là queue time nằm trong
P95 17 giây của tôi. Nói cách khác, hai con số cộng lại thành một bức tranh: 4 đang
chạy, khoảng 40 đang đợi.

**Tôi tin cả hai, cho hai câu hỏi khác nhau.** Hỏi "batch rộng bao nhiêu" thì đọc
`n_busy_slots`; hỏi "hàng đợi sâu bao nhiêu" thì đọc Little's Law. Nếu chỉ nhìn
`n_busy_slots` = 3.98/4 = 100% rồi kết luận "server khoẻ, đang chạy hết công suất" thì
sai hoàn toàn: 100% đó chính là dấu hiệu bão hoà, không phải dấu hiệu tốt.

**Batching mang lại bao nhiêu throughput thật?** Từ CSV, trong cửa sổ 56.4 giây,
`tokens_predicted_total` đi từ 9221 lên 18530, tức **9309 token**, tức **165 tok/s
tổng**. So với 103.5 tok/s của một stream đơn lẻ (`01-quickstart-results.md`), batching
4 slot chỉ cho **1.60x**, không phải 4x.

Tôi thấy phần này đáng chú ý hơn cả con số 3.98. Batching lẽ ra amortize được việc đọc
weights: một lần đọc phục vụ 4 token thay vì 1. Nhưng thời gian mỗi decode step tăng từ
9.66 ms (batch 1) lên 23.8 ms (`n_decode_total` đi từ 2415 lên 4789 trong 56.4 giây,
tức 42.1 step/s), tức chậm đi 2.46x. Hai thứ ăn mất phần lợi:

1. **Prefill bị chèn vào giữa.** `prompt_tokens_total` tăng từ 7020 lên 10030 trong cùng
   cửa sổ, tức 3010 token prompt được xử lý. Continuous batching xen prefill của request
   mới vào giữa các decode step, và prefill là compute-bound: nó cướp thời gian của
   decode.
2. **Attention trên KV cache scale theo batch.** Mỗi sequence có KV riêng, nên phần
   attention không được amortize như phần weights.
