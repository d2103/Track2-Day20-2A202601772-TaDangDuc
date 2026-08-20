# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 166 | 2.89 | 2400 | 3800 | 4600 | 7.3 | 0.0% |
| 50 | 172 | 2.93 | 15000 | 17000 | 18000 | 40.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.01x** (20% of linear) |
| P95 latency | **4.47x** |
| Effective concurrency at 50 users | 40.7 vs `--parallel 4` slots (occupancy/slot ratio 10.17) |

**Saturated.** Throughput delivered only 1.01x for 5x the offered load, and effective concurrency (40.7) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.01x while P95 moved 4.47x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Đọc kết quả của tôi

**Server bão hoà ở đâu đó dưới 10 user, không phải ở 50.**

Con số thuyết phục tôi không phải P95, mà là **RPS gần như đứng yên giữa hai lần chạy**:
2.89 RPS ở 10 user và 2.93 RPS ở 50 user (166 so với 172 request trong 60 giây). Tăng
offered load gấp 5 lần, throughput nhận thêm **1%**. Một server chưa bão hoà thì RPS
phải tăng theo; RPS bị ghim ở một trần cố định là định nghĩa của bão hoà.

Nếu chỉ chạy ở 50 user, tôi sẽ tưởng knee nằm đâu đó giữa 10 và 50. Nhưng Little's Law ở
**10 user** đã tố cáo: effective concurrency = 7.3 trong khi server chỉ có **4 slot**.
Tức ngay ở 10 user đã có khoảng 3 request phải xếp hàng. Trần thật sự nằm quanh **4-6
user đồng thời**, đúng bằng `--parallel 4`. Tôi không đo được chính xác điểm gãy vì lab
chỉ chạy 10 và 50; muốn khoanh vùng thì phải sweep 1/2/4/8 user.

**Phần latency tăng thêm là queue time, không phải compute time.** Bằng chứng từ ba
nguồn độc lập, khớp nhau:

- Little's Law ở 50 user: avg latency 13.89 giây, trong khi 4 slot ở 2.93 RPS chỉ tương
  ứng khoảng 1.37 giây thời gian phục vụ thật. **Khoảng 90% độ trễ là thời gian nằm
  chờ**, không phải thời gian được tính toán.
- `requests_deferred` = **40-45** liên tục (`02-server-batching-u50.md`): server tự khai
  báo có hơn 40 request không tìm được slot khi đến.
- Bản thân compute không hề chậm đi: `n_busy_slots_per_decode` = 3.98/4, tức 4 slot chưa
  bao giờ rảnh. Server không yếu đi, nó chỉ hết chỗ.

**Goodput@SLO.** Đặt SLO P95 nhỏ hơn hoặc bằng 5 giây, mức tôi cho là chấp nhận được với
một trợ lý hỏi-đáp:

| | P95 | Đạt SLO? | Goodput |
|:--|--:|:--|--:|
| 10 user | 3.8 s | có | **2.89 RPS** |
| 50 user | 17.0 s | không | **0 RPS** |

Đây là điểm mấu chốt. Xét theo throughput thô thì hai lần chạy như nhau (2.89 so với
2.93), nhưng xét theo goodput thì lần 50 user **vô giá trị**: mọi request đều về đích
quá muộn. Zero failure không cứu được gì. Locust báo 0% lỗi ở cả hai lần, nên nếu chỉ
nhìn cột failure tôi sẽ kết luận sai rằng server vẫn ổn ở 50 user.

**Knob tôi sẽ chỉnh đầu tiên: nâng `--parallel` từ 4 lên 8.**

Vì bottleneck hiện tại là **số slot**, không phải tốc độ tính toán. Tôi biết vậy nhờ
`n_busy_slots` chạm trần 4 trong khi 40 request đứng đợi: thêm chỗ ngồi chính xác là thứ
đang thiếu. Ngoài ra, decode ở batch nhỏ trên GPU bị chặn bởi băng thông VRAM, nên mỗi
decode step rộng hơn sẽ amortize việc đọc weights trên nhiều token hơn, một lần đọc
phục vụ 8 token thay vì 4.

Tôi kỳ vọng mức lợi **dưới 2x**, không phải đúng 2x, vì đo ở trên cho thấy đi từ batch 1
lên batch 4 chỉ được 1.60x (165 so với 103.5 tok/s): phần attention trên KV cache và
phần prefill xen giữa không amortize được.

**Ràng buộc phải canh:** RTX 3060 Laptop chỉ có 6 GB VRAM, weights đã chiếm 2.97 GB.
`ctx=2048` chia cho 4 slot đang cho 512 token mỗi slot; lên 8 slot mà giữ nguyên `ctx`
thì mỗi slot còn 256 token, quá ngắn cho query RAG. Nên phải nâng `LAB_N_CTX` cùng lúc,
và KV cache lớn hơn phải vừa với khoảng 3 GB VRAM còn lại. Đây là đánh đổi thật, không
phải bật một cờ là xong.

**Knob tôi sẽ KHÔNG chỉnh: hạ quantization.** `01-quickstart-results.md` cho thấy trên
GPU, 2-bit không nhanh hơn 4-bit (1.01x) mà chất lượng trả lời kém hơn rõ. Ở đây nó chỉ
giải phóng VRAM, hữu ích để nuôi KV cache cho nhiều slot hơn, nhưng đó là tác dụng gián
tiếp chứ không sửa đúng bottleneck.
