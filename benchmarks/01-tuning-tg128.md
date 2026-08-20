# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 117.0 | 100% |
| 4 | 116.1 | 99% |
| 8 | 115.7 | 99% |
| 16 | 115.9 | 99% |
| 32 | 115.1 | 98% |

**Best**: `-t 1` at 117.0 tok/s
**Slowest tested**: `-t 32` at 115.1 tok/s (1.02x spread)
**Against the physical-core default** (`-t 8`, 115.7 tok/s): 1.01x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Giải thích của tôi

**Không có knee. Đường cong phẳng, và đó mới là kết quả.**
Spread từ `-t 1` đến `-t 32` chỉ 1.02x (117.0 xuống 115.1 tok/s). Chênh lệch giữa `-t 8`
(số physical core) và `-t 16` (số logical core) là 0.2%. Con số đó nhỏ hơn nhiễu giữa
các lần chạy, nên tôi **không** đọc nó như một xu hướng.

**Vì sao phẳng: sweep này chạy với `ngl=99`.** Toàn bộ layer đã nằm trên RTX 3060, nên
`tg128` (vốn đo thuần decode) đang đo GPU chứ không đo CPU. Cờ `-t` chỉ điều khiển
threadpool phía CPU, mà phần việc còn lại cho CPU ở chế độ này gần như bằng không:
tokenize, sampling, điều phối. Tăng thread cho một thành phần không phải bottleneck thì
không thể nhanh hơn. Đây đúng ra là kết quả *nên* kỳ vọng, chỉ là deck trình bày thread
sweep trong ngữ cảnh CPU inference.

**Bằng chứng cho lập luận trên nằm ở một file khác.** Trong `01-quickstart-cpu-ngl0.md`
tôi chạy lại đúng model đó với `ngl=0`: decode rơi từ 103.5 xuống 11.7 tok/s. Khi CPU
thật sự phải làm việc, nó chỉ đạt khoảng 1/9 tốc độ. Toàn bộ 115-117 tok/s trong bảng
trên là công của GPU, và `-t` không chạm được vào đó.

**Tôi KHÔNG dùng gợi ý `LAB_N_THREADS=1` mà file này tự sinh ra.** Ở chế độ `ngl=99`,
việc `-t 1` "thắng" 1.01x là nhiễu, không phải phát hiện. Và `tg128` là bài đo **một
stream**; khi server chạy `--parallel 4` dưới 50 user đồng thời, CPU còn phải tokenize
và điều phối cho hàng chục request cùng lúc. Bóp xuống 1 thread ở đó là tự tạo ra một
bottleneck mới mà bài đo này không nhìn thấy. Tôi giữ mặc định `-t 8`, bằng số physical
core.

**Thí nghiệm tiếp theo nếu có thêm thời gian:** sweep lại `-t` với `ngl=0`. Đó mới là
điều kiện để knee xuất hiện, và tôi kỳ vọng thấy nó quanh 8 thread rồi tụt khi lên 16 vì
các hyperthread tranh nhau cùng một đường băng thông RAM. Tôi chưa chạy nên không khẳng
định.
