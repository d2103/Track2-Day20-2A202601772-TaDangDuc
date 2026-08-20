# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 3410.2 | 3410.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 3061.7 | 3061.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 3025.2 | 3025.4 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **3165.7** · total **3165.9**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.

## N16-N19: cái nào real, cái nào stub

| Day | Piece | Real hay stub | Cụ thể là gì |
|:--|:--|:--|:--|
| N16 | Cloud / IaC | **stub** | Không có. Mọi thứ chạy local trên laptop, không Terraform, không cloud resource. |
| N17 | Data pipeline | **stub** | Không có ingest hay transform. Corpus là `TOY_DOCS` hard-code trong `pipeline.py`. |
| N18 | Lakehouse | **stub** | Không có bảng, không có storage layer. |
| N19 | Vector + features | **stub** | Retrieval là **keyword overlap**, không phải vector search. Không embedding model, không index. Đó là lý do cột embed = **0.0 ms**: không có phép tính nào xảy ra, chứ không phải vì nó nhanh. |
| N20 | Serving | **real** | `llama-server` (llama.cpp b10488) thật, Gemma 4 E2B UD-Q4_K_XL, `ngl=99` trên RTX 3060. Ba câu trả lời ở trên là output thật của model. |

Tôi giữ nguyên hai hàm `STUB` trong `labs/03-integrate/pipeline.py`.

**Stage chiếm nhiều nhất: `llm`, 3165.7 ms trên tổng 3165.9 ms, tức 100.0%.**
embed 0.0 ms, retrieve 0.1 ms, llm 3165.7 ms.

**Có đúng kỳ vọng không? Đúng về hướng, nhưng con số 100% là do stub thổi phồng.**

Tôi kỳ vọng LLM chiếm đa số, và nó đúng vậy. Nhưng "100%" ở đây không phải một phát hiện
về RAG: nó là hệ quả của việc retrieve chỉ quét 6 document trong RAM bằng phép so khớp
từ khoá. Một N19 thật với vector index trên hàng triệu chunk sẽ tốn hàng chục ms cho
embed và hàng chục ms cho ANN search. Ngay cả khi đó, cộng lại vẫn chỉ khoảng 2-3% của
3.1 giây LLM. **Kết luận không đổi, nhưng tỉ lệ thì đổi**, và tôi sẽ không trích dẫn con
số 100% như thể nó đại diện cho một pipeline production.

**Nếu phải giảm latency pipeline này đi một nửa, tôi tấn công vào decode.**

3.1 giây gần như toàn bộ là thời gian sinh token, không phải prefill: prompt ở đây rất
ngắn, và baseline của tôi cho TTFT P50 chỉ 648 ms trong khi TPOT là 9.66 ms/token. Ở tốc
độ đó, 3.1 giây tương đương khoảng 250-300 token output. Ba hướng, theo thứ tự tôi sẽ
thử:

1. **Giới hạn độ dài output.** Câu trả lời cho ba câu hỏi này chỉ cần 2-3 câu. Đây là
   cách duy nhất trong ba cách cho đúng 2x ngay lập tức mà không đổi phần cứng: sinh nửa
   số token thì tốn nửa thời gian, vì decode tuyến tính theo số token.
2. **Stream về client.** Không giảm tổng thời gian, nhưng người dùng thấy token đầu sau
   648 ms thay vì nhìn màn hình trắng 3.1 giây. Latency cảm nhận giảm khoảng 5x.
3. **Prompt caching cho phần context lặp lại.** Ba query dùng chung vài document nên
   phần prefix của prompt trùng nhau. Cái này chỉ cắt vào TTFT, mà TTFT chỉ chiếm khoảng
   20%, nên nó là hướng yếu nhất ở đây, dù sẽ là hướng mạnh nhất nếu context RAG dài hơn
   nhiều.

Điều tôi sẽ **không** làm là tối ưu retrieve. Nó đang tốn 0.1 ms; có làm nó nhanh vô hạn
thì tổng thời gian cũng chỉ giảm 0.003%.
