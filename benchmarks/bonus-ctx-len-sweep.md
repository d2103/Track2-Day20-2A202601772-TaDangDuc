# Bonus - Context-length sweep (prefill cost)

Host `Windows-AMD64` · llama.cpp `b10488` ·
`threads=8` `ngl=99` · RAM 15.4 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 1451.8 | 176.3 | 1.00x |
| 1024 | 2436.0 | 420.4 | 0.60x |
| 2048 | 3661.0 | 559.4 | 0.40x |
| 4096 | 4100.8 | 998.8 | 0.35x |
| 8192 | 4147.5 | 1975.2 | 0.35x |

At 8192 tokens, prefill costs **1975 ms**, which is
**0.35x** linear scaling -- so on this hardware, over this range, prefill is
still growing **roughly linearly**, not quadratically.

That is the correct finding, not a failed experiment. Attention is O(N^2), but it is only
one term: the per-layer linear projections and MLP are O(N), and on a 2B-class model at
short prompts they dominate. The quadratic term only overtakes them once N gets large
enough. Your prefill cost is currently bounded by throughput, not by sequence length.

To find where it *does* bend, extend the grid:

```bash
.venv/bin/python bonus/sweeps/ctx-len-sweep.py --grid 1024,4096,8192,16384,32768
```

Watch the "vs linear" column: the first row that climbs meaningfully above 1.0 is where
attention starts to matter on your machine. Report that crossover point.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Phát hiện của tôi

**Prefill trên máy tôi không siêu tuyến tính. Nó *dưới* tuyến tính, và càng prompt dài
càng rẻ trên mỗi token.** Ở 8192 token, prefill tốn 1975 ms, chỉ bằng **0.35x** những gì
scaling tuyến tính từ điểm 256 token dự đoán.

Thứ tôi thấy rõ nhất không nằm ở cột thời gian mà ở cột **throughput**: nó *tăng* từ
1451.8 lên 4147.5 tok/s, tức **2.86x**, khi prompt dài ra 32 lần. Nếu attention O(N^2)
đang chi phối thì con số này phải đi xuống.

**Cơ chế: ở prompt ngắn, GPU đang bị bỏ đói.** Prefill là một phép nhân ma trận theo
batch trên toàn bộ prompt cùng lúc, nên 256 token là một batch quá nhỏ để lấp kín RTX
3060 — phần chi phí cố định mỗi lần gọi (launch kernel, đồng bộ, đọc weights của từng
layer) bị chia cho quá ít token. Kéo prompt dài ra chính là làm cho batch đó to lên: cùng
một lần đọc weights nay phục vụ nhiều token hơn. Đây đúng là cơ chế amortize mà tôi đã
gặp ở phần base khi batching 4 slot cho 1.60x throughput, chỉ khác là ở đây các "slot"
nằm trong cùng một prompt.

**Nhưng phần lợi đó đã cạn.** Nhìn tỉ lệ giữa các lần gấp đôi liên tiếp:

| Đoạn | Thời gian tăng | Ý nghĩa |
|:--|--:|:--|
| 256 → 1024 (4x) | 2.39x | rất dưới tuyến tính, GPU còn trống nhiều |
| 1024 → 2048 | 1.33x | vẫn dưới tuyến tính |
| 2048 → 4096 | 1.79x | gần chạm tuyến tính |
| 4096 → 8192 | **1.98x** | **đã tuyến tính** |

Throughput giữa 4096 và 8192 chỉ nhích 4100.8 lên 4147.5 tok/s, tức **1.1%**: GPU đã bão
hoà tính toán. Từ đây trở đi mỗi lần gấp đôi prompt sẽ tốn gấp đôi thời gian, và điểm mà
số hạng O(N^2) của attention bắt đầu vượt lên phải nằm **sau 8192 token**, ngoài dải tôi
đo. Tôi không khẳng định đã thấy đường cong bậc hai, vì tôi chưa thấy.

**RAG pipeline của tôi gánh được bao nhiêu chunk?**

Ở 4147 tok/s trong vùng bão hoà, mỗi chunk 512 token tốn thêm khoảng **123 ms TTFT**, trả
đủ trên **mọi** request. Baseline của tôi có TTFT P50 648 ms với prompt chỉ 11-20 token.
Nếu đặt ngân sách TTFT là 1 giây cho một request đơn lẻ, tôi còn khoảng 350 ms cho
context, tức **khoảng 3 chunk**. Muốn nhồi 16 chunk (8192 token) thì TTFT lên gần 2.6
giây trước khi user thấy token đầu tiên.

Và đó mới là trường hợp một người dùng. Dưới load thì tệ hơn nhiều: ở phần base tôi đo
được server chỉ có 4 slot, xử lý 3010 prompt token trong 56.4 giây, tức khoảng 0.73 giây
prefill cho toàn bộ cửa sổ. Nếu 172 request đó mỗi cái mang 8192 token context, tổng
prefill thành khoảng 1.4 triệu token, tức **hơn 5 phút** công việc prefill nhồi vào một
cửa sổ 56 giây. Server sẽ sụp, không phải chậm đi.
