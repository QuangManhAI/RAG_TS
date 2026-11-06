# laya-jev.md — Kết quả đưa Laya vào RAG theo hướng Jev

- **Motivation/Background**: Nhóm muốn dùng mô hình quyết định kiểu Jev để xếp hạng tài liệu và giảm ảo giác trong RAG. Môn học yêu cầu chạy bằng TS/JS, nên Laya chạy local trên Node được chọn thay cho Jev.
- **Purpose**: Ghi lại Jev và Laya là gì, cách tích hợp, kết quả đo, giới hạn và việc còn lại.
- **Overview Pipeline**: Câu hỏi → vector search → Laya chấm → trộn RRF → cổng bằng chứng → LLM → cổng groundedness → trả lời hoặc từ chối.
- **Detailed Plan**: §1 Kết luận; §2 Jev và Laya; §3 Kiến trúc; §4 Kết quả đo; §5 Chạy thật; §6 Giới hạn; §7 Cấu hình; §8 Chạy lại bài đo; §9 Nguồn; §10 Việc còn lại.
- **References**: [JEV.jpeg](JEV.jpeg), [bài MindStudio](jev-steerable-reranker-mindstudio.pdf), [backend_rag/src/engines/judge/judge.service.ts](../../backend_rag/src/engines/judge/judge.service.ts), [backend_rag/src/modules/rag/rag.service.ts](../../backend_rag/src/modules/rag/rag.service.ts), [backend_rag/benchmark/laya-eval-set.json](../../backend_rag/benchmark/laya-eval-set.json), [docs/AI_chat_log/Vuong_chat_logs.md](../AI_chat_log/Vuong_chat_logs.md) (D-058, D-059).
- **Created**: 2026-11-05T03:47:43+07:00
- **Last Updated**: 2026-11-06T13:13:54+07:00

[STATUS: IN_PROGRESS]

---

## 1. Kết luận

- **Nên giữ Laya, với vai trò trọng tài cho RAG.** Laya không thay được bước tìm kiếm, và dùng riêng để xếp hạng thì kém vector search.
- **Cấu hình tốt nhất đã đo:** vector + Laya trộn RRF trên top 10: MRR **0,577**, so với 0,494 của vector.
- **Cổng bằng chứng hữu ích:** ở ngưỡng 0,6, giữ 27/28 câu trả lời được và chặn 11/14 câu không có đáp án.
- **Cổng groundedness còn yếu:** ở ngưỡng 0,3, giữ 10/10 câu đúng nhưng chỉ chặn 3/10 câu bịa.
- **Độ tin cậy còn hạn chế:** chênh lệch hạng 1 chỉ là 13 so với 11 câu. Bộ test nhỏ, do agent tự viết và cũng được dùng để chọn cấu hình.
- **Jev chưa được đo:** toàn bộ số đo trong tài liệu này là của Laya. Không có so sánh trực tiếp Jev với Laya.

---

## 2. Jev và Laya

| Tiêu chí | Jev | Laya |
| :--- | :--- | :--- |
| **Loại model** | Mô hình quyết định có kiểu | Mô hình quyết định có kiểu |
| **Câu hỏi** | Yes/no, chọn một, chấm điểm | `noul`, `choice`, `score` |
| **Nơi chạy** | API đám mây, cần `TYPESAFE_API_KEY` | Local, không tốn phí API |
| **Trong dự án** | Chỉ tham khảo, không chạy | Bản tiếng Anh, 421M tham số |

Cả hai không sinh văn bản. Chúng nhận nội dung và câu hỏi, rồi trả về xác suất. Cách gọi giống nhau không có nghĩa là chất lượng giống nhau.

Model thực tế: ONNX `receptron/laya-onnx` (commit `d42786b`), chạy bằng npm `@receptron/laya` 0.2.0 và `onnxruntime-node`. Đây là bản xuất của bên thứ ba từ `convaiinnovations/laya`. Dự án không dùng Python.

---

## 3. Kiến trúc đã làm

```text
Câu hỏi
  → Vector search (Qdrant) → lấy nội dung từ rag_ts_corpus
  → Xếp theo similarity + authority → lấy top 10
  → Laya chấm từng chunk → trộn RRF → giữ top_k
  → Cổng bằng chứng (điểm cao nhất trong top_k)
      ├ Đủ → LLM → Cổng groundedness (tối đa 3 chunk đầu)
      │        ├ Đạt → trả lời
      │        └ Không đạt → thay bằng câu từ chối (SSE answer_replaced)
      └ Thiếu → LLM viết lại câu hỏi 1 lần → tìm lại → vẫn thiếu thì từ chối
```

| Vị trí | Laya được hỏi gì | Đánh giá |
| :--- | :--- | :--- |
| **Rerank** | Đoạn văn có trả lời trực tiếp câu hỏi không? | Giữ, chỉ dùng khi trộn RRF |
| **Cổng bằng chứng** | Dùng lại điểm rerank, lấy điểm cao nhất trong context | Giá trị nhất |
| **Groundedness** | Câu trả lời có được đoạn bằng chứng hỗ trợ không? | Giữ tạm, chưa nên tin nhiều |

Nếu Laya lỗi, quá tải hoặc quá 30 giây, chat vẫn chạy như không có Laya. Không đặt `LAYA_MODEL_DIR` thì toàn bộ phần này tắt.

Vì Laya chỉ đọc được 512 token, mỗi chunk được chấm riêng thay vì gộp vào một lần. Kiểu rerank `choice` không phù hợp với dữ liệu này: giới hạn 192 token cho phần lựa chọn chỉ còn khoảng 19 token cho mỗi chunk khi có 10 chunk.

Việc tích hợp còn sửa một lỗi có sẵn: Qdrant đã bỏ `text`, nên trước đó mọi chunk bị loại và app luôn trả lời "không đủ tài liệu". Backend giờ đọc nội dung từ `rag_ts_corpus`.

---

## 4. Kết quả đo

Bộ test gồm 28 câu có đáp án, 14 câu không có đáp án và 10 cặp câu trả lời đúng/sai. Câu hỏi được viết từ các chunk lấy ngẫu nhiên trong corpus, với mỗi câu chỉ một chunk được tính là đúng. Bài đo tắt authority boost.

Chạy lại lần hai cho kết quả giống hệt: 28/28 bộ chunk trùng và điểm không lệch. Điều đó cho thấy bài đo tái lập được, chứ không làm kết luận chắc thêm.

### 4.1. Xếp hạng

| Cách xếp hạng | hit@1 | hit@5 | MRR |
| :--- | :--- | :--- | :--- |
| Vector, top 10 | 39,3% | 67,9% | 0,494 |
| Laya một mình, top 10 | 32,1% | 64,3% | 0,457 |
| **Vector + Laya RRF, top 10 (đang dùng)** | **46,4%** | **71,4%** | **0,577** |
| Vector, top 20 | 39,3% | 67,9% | 0,498 |
| Laya một mình, top 20 | 35,7% | 64,3% | 0,451 |
| Vector + Laya RRF, top 20 | 42,9% | 71,4% | 0,553 |

Chunk đúng nằm trong top 10 của vector ở 20/28 câu và trong top 40 ở 24/28 câu. Rerank không cứu được phần còn lại.

### 4.2. Cách đặt câu hỏi cho Laya (top 10)

| Cách hỏi | hit@1 | hit@5 | MRR |
| :--- | :--- | :--- | :--- |
| `noul`, prompt của dự án | 9/28 | 18/28 | 0,457 |
| `noul`, prompt kiểu TypedRank | 9/28 | 19/28 | 0,446 |
| `score` 0–3 | 8/28 | 19/28 | 0,426 |
| Vector + `noul` dự án (RRF) | 13/28 | 20/28 | 0,577 |
| Vector + `noul` TypedRank (RRF) | 13/28 | 20/28 | 0,577 |
| Vector + `score` 0–3 (RRF) | 13/28 | 19/28 | 0,570 |

Đổi cách hỏi không vượt được vector khi dùng riêng. Trộn với vector mới tạo ra khác biệt.

### 4.3. Cổng bằng chứng (top 5 sau khi trộn)

| Ngưỡng | Giữ câu trả lời được | Chặn câu không có đáp án |
| :--- | :--- | :--- |
| 0,3 | 28/28 | 7/14 |
| 0,5 | 27/28 | 9/14 |
| **0,6 (đang dùng)** | **27/28** | **11/14** |
| 0,7 | 27/28 | 12/14 |
| 0,8 | 18/28 | 13/14 |

"Giữ" chỉ nghĩa là có chunk đạt ngưỡng, chưa chắc đó là chunk đúng. Ở ngưỡng 0,6 vẫn lọt 3 câu hỏi về model không có thật: `ZephyrNet-9000`, `BlorpGNN` và `HyperLlama-11`.

### 4.4. Cổng groundedness

| Ngưỡng | Giữ câu đúng | Chặn câu bịa |
| :--- | :--- | :--- |
| 0,2 | 10/10 | 1/10 |
| **0,3 (đang dùng)** | **10/10** | **3/10** |
| 0,5 | 8/10 | 7/10 |
| 0,7 | 5/10 | 9/10 |

Có 3/10 cặp mà câu bịa được chấm cao hơn câu đúng, ví dụ 0,74 so với 0,72, nên đổi ngưỡng không tách hết được hai nhóm. Các câu trả lời này ngắn và do agent tự viết; câu trả lời dài của LLM chưa được đo.

### 4.5. Chi phí

| Hạng mục | Số đo |
| :--- | :--- |
| **Một lượt Laya** | Khoảng 426 ms trên CPU, khi máy đang thiếu RAM |
| **Chấm 10 chunk** | Tính ra khoảng 4,3 giây |
| **Tải model** | Khoảng 4,2 giây |
| **RAM** | Tiến trình test khoảng 1,6 GB sau khi load Laya |
| **Ổ đĩa** | `model/laya` 1,6 GB |

---

## 5. Chạy thật qua app

Thử trên cổng 3100 với LLM `qwen2.5-7b-instruct-q4_k_m`. Mới thử 3 câu, chưa có bài đo tải.

| Câu hỏi | Kết quả | Tổng | Laya chấm | LLM |
| :--- | :--- | :--- | :--- | :--- |
| **Số trang web của ClueWeb12** | Đúng 733.019.372 trang, có citation | 32,0 s | 8,4 s | 22,0 s |
| **Thủ đô của Úc** | Từ chối đúng | 16,1 s | 2,6 s | Không sinh câu trả lời |
| **Vai trò của human-in-the-loop trong Saarthi** | Trả lời đúng ý, có citation | 34,2 s | 4,2 s | 26,9 s |

Câu thủ đô Úc tốn thêm 10,5 giây cho bước viết lại câu hỏi. Trước khi trộn RRF, câu ClueWeb12 bị từ chối nhầm vì chunk đúng đứng hạng 6, ngoài top 5.

---

## 6. Giới hạn

- **Bộ test:** 28 câu là ít, chỉ toàn tiếng Anh và hỏi rất cụ thể. Câu hỏi có thể dùng lại từ trong chunk, và chunk khác cũng trả lời được vẫn bị tính sai.
- **Chọn cấu hình trên chính dữ liệu đo:** RRF và các ngưỡng không có tập kiểm tra riêng, nên số đo có thể lạc quan.
- **Khác với app:** bài đo tắt authority boost, trong khi app bật mặc định.
- **Tiếng Việt:** bản ONNX này chỉ có checkpoint tiếng Anh. Câu hỏi tiếng Việt chưa được đo và có thể kém.
- **Độ khớp của bản ONNX:** tác giả ghi lệch logit khoảng 1e-5 so với PyTorch, nhưng dự án chưa kiểm chứng độc lập. Ví dụ trong README cũng cho số khác khi chạy lại (`billing` 0,48 so với 0,94). README rút gọn nội dung ví dụ nên có thể input không giống, nhưng điều này chưa được xác nhận.
- **Fine-tune:** đang dùng bản chưa tinh chỉnh cho paper khoa học, nên kết quả có thể chưa nói hết tiềm năng của Laya.
- **Tìm kiếm:** RAG_TS không có BM25. Elasticsearch chỉ có index `knowledge_chunks` với 128 tài liệu. Sync corpus còn thiếu 827 chunk và ít nhất 11.928 paper.

---

## 7. Cấu hình

Thêm vào `backend_rag/.env`, file không được commit:

```bash
CORPUS_MONGO_URI=mongodb://<user chỉ đọc>:<mật khẩu>@localhost:27017/rag_ts_corpus?authSource=admin&directConnection=true
LAYA_MODEL_DIR=../model/laya
```

| Biến | Mặc định | Ý nghĩa |
| :--- | :--- | :--- |
| `LAYA_RERANK` | `true` | Trộn RRF. `false` thì giữ thứ tự cũ và chỉ chấm top_k cho cổng bằng chứng |
| `LAYA_EVIDENCE_THRESHOLD` | `0.6` | Dưới mức này là thiếu bằng chứng |
| `LAYA_GROUNDED_THRESHOLD` | `0.3` | Dưới mức này thì thay câu trả lời bằng câu từ chối |
| `LAYA_TIMEOUT_MS` | `30000` | Quá hạn thì bỏ qua Laya |

Tài khoản `MONGO_ADMIN_*` có quyền `root`, nên app không được dùng nó (D-056). Backend sẽ log `CORPUS_OVERPRIVILEGED` nếu dùng tài khoản quyền quá cao. Các bài đo trong tài liệu này đã tạm dùng tài khoản đó để đọc, chỉ qua biến môi trường của tiến trình.

---

## 8. Chạy lại bài đo

Chạy từ `backend_rag/`, sau khi đặt `QDRANT_URL`, `QDRANT_API_KEY`, `QDRANT_COLLECTION`, `EMBEDDING_MODEL`, `CORPUS_MONGO_URI` và `LAYA_MODEL_DIR`. Hai script chỉ đọc dữ liệu.

```bash
EVAL_OUT=/tmp/laya-eval.json node benchmark/eval-laya.mjs
EVAL_OUT=/tmp/laya-eval.json node benchmark/eval-laya-variants.mjs
```

Script đầu mất khoảng 5 phút với 720 lượt Laya và ghi kết quả chi tiết ra `EVAL_OUT`. Script thứ hai dùng lại file đó.

Kiểm thử backend gần nhất: 248/248 test đạt. Frontend có 11 test lỗi, vẫn đúng 11 khi bỏ phần sửa `answer_replaced`; lỗi thấy ở các test đã kiểm tra là `localStorage` không có sẵn trên Node 26.

---

## 9. Nguồn

| Nguồn | Đã đọc | Ghi nhận | Lưu ý |
| :--- | :--- | :--- | :--- |
| **Bài MindStudio** | Có | BM25 21% → 54% top-1 khi thêm Jev; khoảng 17 quyết định/giây | Blog có quảng cáo, không nêu bộ dữ liệu; nền BM25 khác hệ vector của dự án |
| **Sơ đồ `JEV.jpeg`** | Có | Hai cổng chống ảo giác | Sơ đồ khái niệm, không có số đo |
| **Model card Laya** | Có | Search relevance accuracy 0,628; routing 0,991; moderation 0,967 | Số do tác giả công bố |
| **TypedRank** | Có (tài liệu, một phần code) | Có `LayaBackend` | Tự ghi benchmark chỉ là smoke test, không dùng để so chất lượng model |
| **`laya-ara-rag`** | Có | Top-1 0,153 → 0,588 sau fine-tune trên MIRACL-ar | Riêng cho tiếng Ả Rập |
| **`layaAsRagJudge`** | Có (README) | Kiểm tra claim: accuracy 75%, xác nhận nhầm 27,8% | Không phải rerank; bộ mẫu 72 claim |
| **Medium, Reddit** | Không | Không có | Không truy cập được, nên chưa đánh giá |

---

## 10. Việc còn lại

| Ưu tiên | Việc | Lý do |
| :--- | :--- | :--- |
| 1 | Tạo tài khoản chỉ đọc cho corpus, thêm 2 biến vào `.env` | App chưa dùng được phần mới nếu thiếu |
| 2 | Bật `backend_rag-mongo-1`, chạy lại BE cổng 3000 | Agent đã tắt nhầm BE này bằng `pkill` khi dọn server test |
| 3 | Chốt ngưỡng bằng chứng 0,6 hay 0,7 | 0,7 chặn thêm 1 câu, nhưng 0,8 giảm mạnh còn 18/28 |
| 4 | Mở rộng bộ test lên 100–200 câu, có câu hỏi thật và tập kiểm tra riêng | Xác nhận mức tăng hiện tại |
| 5 | Thêm BM25 và đo lại | 4/28 câu nằm ngoài top 40; lợi ích còn là giả thuyết |
| 6 | Đo tải bằng k6 với Laya bật | Laya chạy từng việc một, quá 8 yêu cầu chờ thì bỏ qua |
| 7 | Cân nhắc fine-tune Laya | Cần dữ liệu có nhãn và GPU |
| 8 | Commit khi nhóm đồng ý | Code hiện vẫn ở working tree |
