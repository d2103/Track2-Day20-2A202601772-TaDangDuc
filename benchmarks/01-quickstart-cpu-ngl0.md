# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5814 | 1323 / 1633 | 85.5 / 96.7 | 6733 / 7631 / 7631 | 11.7 |
| UD-Q2_K_XL | 2.24 | 5973 | 1436 / 1759 | 72.5 / 75.3 | 6032 / 6272 / 6272 | 13.8 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.18x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Nhận xét của tôi (run đối chứng CPU-only)

File này **không phải** baseline của lab. Đây là lần chạy đối chứng với
`LAB_N_GPU_LAYERS=0`, tức ép toàn bộ decode xuống CPU. Baseline thật (`ngl=99`) nằm ở
`01-quickstart-results.md`. Tôi chạy nó vì hai lý do:

**1. Nó là "before" cho REFLECTION §5.** Cùng model, cùng quantization, cùng prompt set,
chỉ đổi một cờ:

| | ngl=0 (CPU) | ngl=99 (GPU) | tỉ lệ |
|:--|--:|--:|--:|
| Decode UD-Q4_K_XL | 11.7 tok/s | 103.5 tok/s | **8.85x** |
| TPOT P50 | 85.5 ms | 9.66 ms | **8.85x** |
| E2E P50 | 6733 ms | 1250 ms | 5.39x |
| TTFT P50 | 1323 ms | 648 ms | 2.04x |

**2. Nó giải thích một chuyện mà baseline GPU không giải thích được.** Trên GPU, bản
2-bit gần như không nhanh hơn bản 4-bit (1.01x). Nhưng ở đây, trên CPU, **chính hai file
weights đó cho 1.18x** (13.8 vs 11.7 tok/s; TPOT 72.5 vs 85.5 ms).

Cùng số byte, cùng model, kết quả khác nhau, tức **nút cổ chai đã đổi chỗ** chứ không
phải bản thân quantization vô dụng. Trên CPU, decode bị chặn bởi băng thông RAM: DDR4
dual-channel của Ryzen 5800H cho khoảng 51 GB/s, và 2.97 GB / 51 GB/s xấp xỉ 17 tok/s
trần lý thuyết. Tôi đo được 11.7, tức khoảng 68% roofline, đúng tầm hiệu suất thực tế
của DDR4. Giảm 25% số byte phải đọc thì nhanh lên thấy rõ, và con số 1.18x (thay vì
1.33x như tỉ lệ dung lượng) là vì "UD" của Unsloth là dynamic quant: không phải tensor
nào cũng bị hạ xuống 2-bit.

Phần phân tích cho GPU tôi viết trong `01-quickstart-results.md`.
