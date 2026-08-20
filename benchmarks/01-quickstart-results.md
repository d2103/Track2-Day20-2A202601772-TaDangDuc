# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 8294 | 648 / 1145 | 9.7 / 10.2 | 1250 / 1745 / 1745 | 103.5 |
| UD-Q2_K_XL | 2.24 | 8228 | 705 / 1002 | 9.6 / 10.2 | 1280 / 1609 / 1609 | 104.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `UD-Q4_K_XL` decode within 2% of each other here, for 0.73 GB difference on disk.

## Nhận xét của tôi

**Kết quả ngắn gọn: 2-bit nhỏ hơn 25% dung lượng nhưng chỉ nhanh hơn 1%.**
UD-Q2_K_XL 2.24 GB vs UD-Q4_K_XL 2.97 GB (-0.73 GB), decode 104.5 vs 103.5 tok/s,
TPOT P50 9.57 vs 9.66 ms. TTFT P50 của bản 2-bit thậm chí *chậm hơn* (705 vs 648 ms).

**Vì sao nó không nhanh hơn như kỳ vọng từ deck.**
Bench chạy với `ngl=99` trên RTX 3060 Laptop, tức weights nằm trong VRAM chứ không
phải RAM hệ thống. Con số ~104 tok/s tự nó đã xác nhận điều đó: RAM DDR4 dual-channel
của Ryzen 5800H chỉ cho khoảng 50 GB/s, không đủ để nuôi tốc độ này ở batch = 1.
Khi đã nằm trên VRAM, decode vẫn bị chặn bởi bandwidth, nên về lý thuyết đọc ít byte
hơn phải nhanh hơn. Hai lý do nó không xảy ra:

1. **"UD" là Unsloth Dynamic** — không phải mọi tensor đều bị hạ xuống 2-bit. Tỉ lệ
   -25% trên đĩa không đồng nghĩa với -25% số byte thực đọc mỗi token.
2. **Q2_K dequantize tốn ALU hơn mỗi byte** (superblock nhỏ hơn, nhiều hệ số scale hơn
   phải giải nén). Phần bandwidth tiết kiệm được bị trả lại bằng compute, và ở kích
   thước model này hai thứ gần như triệt tiêu nhau.

**Vì sao TTFT không phản ánh gì.** Prompt trong bench chỉ 11-20 token. Log server cho
thấy prompt eval thật chỉ 60-230 ms, trong khi TTFT đo phía client là 648 ms — phần
lớn TTFT ở đây là overhead cố định (HTTP, tokenize, chọn slot), không phải công việc
phụ thuộc quantization. Chênh lệch 57 ms giữa hai quant nằm trong nhiễu.

**Có đáng không: không, trên máy này thì không.**
Tôi hỏi cùng một câu (bài toán đọc P95 vs throughput dưới load, `seed=42`,
`max_tokens=220`) cho cả hai bản:

- **Q4_K_XL** hiểu đúng đề, chỉ ra tranh chấp tài nguyên và hàng đợi là nguyên nhân.
- **Q2_K_XL** *đọc sai đề*: nó hiểu "dưới 50 user đồng thời" thành "tải thấp", rồi xây
  toàn bộ phần giải thích trên tiền đề sai đó. Nó cũng tốn nhiều token hơn để nhắc lại
  đề bài trước khi trả lời.
- Tốc độ trong chính lần A/B đó: 107.5 vs 101.8 tok/s — bản 2-bit vẫn **không** nhanh hơn.

Nên đánh đổi ở đây là: mất 0.73 GB dung lượng, được 1% tốc độ, và trả giá bằng một lỗi
đọc hiểu. Đó là lỗ. 2-bit chỉ đáng khi dung lượng là ràng buộc cứng — VRAM 6 GB của tôi
đã chứa vừa bản 4-bit, nên tôi không ở trong tình huống "2-bit hoặc không chạy được".
Nếu model lớn gấp đôi và bản 4-bit tràn VRAM sang RAM, kết luận sẽ đảo ngược hoàn toàn.

*Giới hạn của phép đo:* 10 request mỗi quant, và so sánh chất lượng chỉ trên 1 prompt —
đủ để ra quyết định cho lab này, chưa phải một eval nghiêm túc.
