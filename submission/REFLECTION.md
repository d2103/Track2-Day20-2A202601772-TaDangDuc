# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.

**Họ Tên:** Tạ Đăng Đức
**Cohort:** A20-K3
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

- **OS:** Windows 11 Home Single Language (build 26200)
- **CPU:** AMD Ryzen 7 5800H with Radeon Graphics
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2 (Zen 3). Probe trên Windows không in ra danh sách flag nên
  đây là suy ra từ kiến trúc, không phải số đo.
- **RAM:** 15.4 GB
- **Accelerator:** NVIDIA GeForce RTX 3060 Laptop GPU, 6 GB VRAM — backend **CUDA**,
  `ngl=99` (toàn bộ layer offload lên GPU)
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi. Không dùng Colab hay Kaggle.

**Setup story:** `.\lab.ps1` chết ngay lần chạy đầu với `The '<' operator is reserved`.
Nguyên nhân là encoding, không phải code: file lưu UTF-8 **không BOM**, mà Windows
PowerShell 5.1 decode file không BOM bằng ANSI codepage 1252. Dấu em dash `—` ở dòng 48
biến thành `â€”`, trong đó byte `0x94` thành `”` — PowerShell coi ký tự đó là dấu nháy
đóng, string đứt giữa chừng, cả `switch` block hỏng theo. Tôi lưu lại file kèm BOM là
chạy được. Cùng lỗi này khiến mọi `benchmarks/*.md` do lab sinh ra bị ghi bằng cp1252
(`labkit.write_report` gọi `write_text()` không chỉ định encoding), nên tôi lưu lại tất
cả bằng UTF-8 trước khi push, nếu không GitHub sẽ hiện ký tự lỗi.

Sửa xong encoding thì `verify` lộ ra một bug thứ hai, cũng chỉ xảy ra trên Windows:
`scripts/verify.py` so đường dẫn dạng `benchmarksile.md` (backslash, do `str()` trên
WindowsPath) với output của `git ls-files` vốn luôn dùng forward slash, nên **mọi file
trong thư mục con đều bị báo là chưa commit** kể cả sau khi đã commit thật. Tôi sửa một
dòng, đổi `str(...)` thành `.as_posix()`, có ghi comment tại chỗ. Fix này là no-op trên
macOS/Linux nên không làm thay đổi kết quả chấm trên máy grader. Ngoài ra `verify.py`
in ký tự `✓` và đọc file bằng locale cp1252 nên crash trên console Windows; cái này tôi
không sửa file, chỉ chạy với `PYTHONUTF8=1`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 8294 | 648 / 1145 | 9.7 / 10.2 | 1250 / 1745 / 1745 | 103.5 |
| UD-Q2_K_XL | 2.24 | 8228 | 705 / 1002 | 9.6 / 10.2 | 1280 / 1609 / 1609 | 104.5 |

**Quan sát:** 2-bit nhỏ hơn 25% dung lượng nhưng chỉ nhanh hơn **1%** (104.5 vs 103.5
tok/s), TTFT P50 còn chậm hơn. Tôi đã hỏi cùng một câu cho cả hai bản qua `serve` và
`serve --compare`: bản 4-bit hiểu đúng đề, bản 2-bit **đọc sai đề** — nó hiểu "dưới 50
user đồng thời" thành "tải thấp" rồi lập luận trên tiền đề sai đó. Mất 0.73 GB, được 1%
tốc độ, trả giá bằng chất lượng: **không đáng** trên máy này. Chi tiết cơ chế trong
`benchmarks/01-quickstart-results.md`.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 2.89 | 2400 | 3800 | 4600 | 7.3 | 0% |
| 50 | 2.93 | 15000 | 17000 | 18000 | 40.7 | 0% |

- **Offered load tăng 5×, throughput thực tăng:** _1.01×_
- **P95 tăng:** _4.47×_
- **Effective concurrency ở 50 users:** _40.7_ so với `--parallel` = _4_ slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `metrics` chạy chồng với `load-50`):
_3.98_ / _4_ slots (100%)

**Saturation reading:** Server bão hoà **dưới 10 user**, không phải ở 50. Bằng chứng
thuyết phục tôi không phải P95 mà là **RPS đứng yên**: 2.89 rồi 2.93 — offered load gấp
5 lần, throughput thêm 1%. Ngay ở 10 user, Little's Law đã cho 7.3 request in-flight
trên 4 slot, tức đã có hàng đợi. Phần latency tăng thêm là **queue time**: avg latency
13.89 s ở 50 user trong khi thời gian phục vụ thật chỉ ~1.37 s, và server tự khai
`requests_deferred` = 40-45. Compute không chậm đi, nó chỉ hết chỗ. Với SLO P95 ≤ 5 s,
goodput đi từ 2.89 RPS xuống **0**. Knob tôi chỉnh đầu tiên là `--parallel` 4 → 8, vì
bottleneck là số slot chứ không phải tốc độ tính toán, và batch rộng hơn amortize được
việc đọc weights trên nhiều token hơn — nhưng phải nâng `LAB_N_CTX` cùng lúc, vì 6 GB
VRAM đã mất 2.97 GB cho weights.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không có | **stub** — chạy local, không cloud resource |
| N17 Data pipeline | không có | **stub** — corpus là `TOY_DOCS` hard-code |
| N18 Lakehouse | không có | **stub** — không bảng, không storage layer |
| N19 Vector + features | keyword overlap | **stub** — không embedding model, không vector index |
| N20 Serving | `llama-server` | **real** |

**Latency split** (mean của 3 query):

- embed: _0.0 ms_
- retrieve: _0.1 ms_
- llm: _3165.7 ms_
- **stage chiếm nhiều nhất:** _llm_ (_100.0%_ của total)

**Reflection:** Bottleneck là decode, đúng như kỳ vọng — nhưng con số 100% là do stub
thổi phồng: retrieve chỉ quét 6 document trong RAM, và embed = 0.0 ms vì **không có phép
tính nào xảy ra**, không phải vì nó nhanh. Một N19 thật vẫn chỉ chiếm 2-3% của 3.1 giây.
Muốn giảm một nửa, tôi cắt số token sinh ra: 3.1 s ở TPOT 9.66 ms tương đương ~300 token
output, mà ba câu hỏi này chỉ cần 2-3 câu trả lời.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

**Change:** bật GPU offload — `LAB_N_GPU_LAYERS=0` → `ngl=99` (toàn bộ layer lên RTX
3060), đo lại bằng chính `benchmark.py` với cùng model, cùng quantization, cùng prompt
set. Bằng chứng: `benchmarks/01-quickstart-cpu-ngl0.md` (before) và
`benchmarks/01-quickstart-results.md` (after).

```
before:  11.7 tok/s   (TPOT P50 85.5 ms,  E2E P50 6733 ms)   ngl=0
after:  103.5 tok/s   (TPOT P50 9.66 ms,  E2E P50 1250 ms)   ngl=99
speedup: 8.85×        (E2E 5.39×)
```

**Tại sao nó work:**

Decode ở batch = 1 bị chặn bởi **băng thông bộ nhớ**, không phải FLOPs: mỗi token sinh
ra phải đọc lại gần như toàn bộ weights một lượt, còn lượng tính toán trên mỗi byte đọc
được thì rất nhỏ. Nên thứ quyết định tốc độ là weights **nằm ở đâu**, chứ không phải CPU
mạnh cỡ nào. DDR4 dual-channel của 5800H cho khoảng 51 GB/s; GDDR6 192-bit của RTX 3060
Laptop cho khoảng 336 GB/s. Tỉ lệ ~6.6×, và tôi đo được 8.85× — cùng bậc độ lớn, phần dư
đến từ chỗ CPU còn phải chia băng thông đó với OS và phần còn lại của hệ thống, trong khi
GPU độc chiếm VRAM của nó. Phép thử roofline khớp: 2.97 GB / 51 GB/s ≈ 17 tok/s trần lý
thuyết trên CPU, tôi đo 11.7 tức ~68% roofline, đúng tầm hiệu suất thực tế của DDR4.

Điều tôi thấy giá trị nhất không phải con số 8.85×, mà là nó **giải thích ngược lại hai
kết quả bất thường** trước đó của tôi:

1. **2-bit không nhanh hơn 4-bit trên GPU (1.01×)** — lúc đầu tôi tưởng quantization vô
   dụng. Nhưng khi ép cùng hai file weights đó xuống CPU, 2-bit **nhanh hơn 1.18×**.
   Cùng số byte, cùng model, kết quả khác nhau, nghĩa là bottleneck đã đổi chỗ: trên CPU
   ta thiếu băng thông nên đọc ít byte hơn là thắng; trên GPU băng thông đã dư nên tiết
   kiệm byte không mua thêm được gì, mà Q2_K lại tốn ALU hơn khi dequantize.
2. **Thread sweep phẳng lì (1.02× từ `-t 1` đến `-t 32`)** — vì với `ngl=99` thì `-t`
   đang điều khiển một thành phần không phải bottleneck. CPU chỉ còn tokenize và
   sampling. Đây là lý do tôi **không** dùng gợi ý `LAB_N_THREADS=1` mà `tune` tự sinh
   ra: 1.01× đó là nhiễu.

Chỗ khác với kỳ vọng từ deck: deck trình bày thread count như một knob đáng tune, và
quantization thấp hơn như một cách đổi chất lượng lấy tốc độ. Trên máy này **cả hai đều
gần như vô hiệu**, vì cả hai đều giả định bottleneck nằm ở CPU. Khi weights đã ở VRAM,
knob duy nhất còn ý nghĩa là số slot (`--parallel`) — và đó đúng là thứ chặn tôi ở phần
load test.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

**Đã làm:** B2 — `sweep-ctx`, đo chi phí prefill theo độ dài prompt
(`benchmarks/bonus-ctx-len-sweep.md`). Đây là knob khác hẳn §5: §5 đổi *nơi* weights nằm,
phần này giữ nguyên mọi thứ và chỉ đổi *độ dài prompt*.

**Numbers:**

```
change:  prompt length 256 → 8192 token (ngl=99, threads=8, cùng model UD-Q4_K_XL)
before:  1451.8 tok/s prefill  (256 token,  TTFT contribution 176.3 ms)
after:   4147.5 tok/s prefill  (8192 token, TTFT contribution 1975.2 ms)
speedup: 2.86× throughput prefill — chi phí mỗi token giảm 2.86 lần
         tổng thời gian chỉ bằng 0.35× mức scaling tuyến tính dự đoán
```

**Điều này nói lên gì mà deck chưa nói:**

Deck dạy attention là O(N²) và dùng điều đó để biện minh cho disaggregated
prefill/decode. Trên máy tôi, trong dải 256-8192 token, **prefill chạy dưới tuyến tính**:
prompt dài gấp 32 lần nhưng throughput lại *tăng* 2.86×. Lý do là O(N²) chỉ là một số
hạng; các phép chiếu tuyến tính và MLP là O(N), và trên model 2B ở prompt ngắn chúng áp
đảo. Quan trọng hơn: ở 256 token, GPU đang bị bỏ đói — batch quá nhỏ để lấp kín RTX 3060,
nên chi phí cố định mỗi lần gọi kernel bị chia cho quá ít token. Kéo prompt dài ra chính
là làm batch to lên.

Đây là cùng một cơ chế amortize mà tôi đã đo ở phần base khi 4 slot batching cho 1.60×
throughput — chỉ khác là ở đây các "slot" nằm bên trong một prompt duy nhất. Nói cách
khác, prefill dài và continuous batching đang mua cùng một thứ: nhiều token hơn cho mỗi
lần đọc weights.

Nhưng phần lợi đó có đáy. Tỉ lệ giữa các lần gấp đôi liên tiếp là 1.33× → 1.79× →
**1.98×**, và throughput giữa 4096 và 8192 chỉ nhích 1.1%. GPU đã bão hoà tính toán, nên
từ 8192 trở đi mỗi lần gấp đôi prompt sẽ tốn đúng gấp đôi thời gian. Điểm mà O(N²) thật
sự vượt lên nằm **ngoài dải tôi đo** — tôi không tuyên bố đã nhìn thấy đường cong bậc
hai, vì tôi chưa thấy.

Hệ quả thực tế mà tôi rút ra cho chính pipeline RAG ở §4: mỗi chunk 512 token tốn thêm
~123 ms TTFT, trả đủ trên **mọi** request. Với ngân sách TTFT 1 giây tôi chỉ đủ chỗ cho
khoảng 3 chunk. Và dưới load thì con số này không phải chuyện của riêng request đó: ở §3
tôi đo được server chỉ có 4 slot và xử lý 3010 prompt token trong 56.4 giây. Nếu 172
request đó mỗi cái mang 8192 token context, tổng prefill thành ~1.4 triệu token, tức hơn
5 phút công việc nhồi vào cửa sổ 56 giây. Prompt dài không chỉ làm chậm chính nó — nó
giữ slot lâu hơn và làm sâu thêm hàng đợi của tất cả những người phía sau. Đó là câu trả
lời cụ thể cho đề xuất "cứ nhồi thêm context vì context window cho phép".

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Rằng hai knob mà lab dạy kỹ nhất — thread count và quantization — lại là hai knob **không
làm được gì** trên máy tôi, và phải chạy một thí nghiệm thứ ba (ngl=0) mới hiểu vì sao.
Một kết quả "không có gì thay đổi" chỉ có nghĩa khi biết bottleneck đang nằm ở đâu.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`tune`)
- [x] `benchmarks/02-server-results.md` committed (`load-report`)
- [x] `benchmarks/02-server-batching-u50.md` + `-metrics-u50.csv` committed (`metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed
- [x] `benchmarks/03-integration-results.md` committed (`pipeline`)
- [x] Mọi section **"required — replace this line"** đã được thay bằng nhận xét của tôi
- [x] 5 screenshots trong `submission/screenshots/` (tôi có 6)
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/`
