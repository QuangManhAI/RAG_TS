# Báo cáo Prometheus — Chủ nhật 08/11/2026, 10:00–12:00 và 20:00–21:00 (+07:00)

- **Motivation/Background**: Chủ nhật 08/11/2026 `backend_rag` nhận traffic người dùng thật (đăng ký, chat RAG, đọc paper) vào buổi sáng và buổi tối. Đây là lần đầu có số liệu vận hành từ Prometheus cho cả một buổi, khác với các lần đo bằng k6 trong `backend_rag/benchmark/`.
- **Purpose**: Tổng hợp và phân tích metric Prometheus của `backend_rag` trong hai khung 10:00–12:00 (§1–§11) và 20:00–21:00 (§12), so sánh hai khung (§13), chỉ ra nút thắt, lỗi và các điểm hở về đo lường, kèm việc cần làm.
- **Overview Pipeline**: Gọi HTTP API của Prometheus (`localhost:9090`, `query` và `query_range`, bước 15 s trên dữ liệu scrape 5 s) → tính số request đúng từ giá trị counter thô, tính phân vị từ bucket histogram → đối chiếu với code (`prometheus-metrics.ts`, `client-slots.ts`, `throttler.ts`, `env.schema.ts`, `.env`) → viết báo cáo.
- **Detailed Plan**: §1 Phạm vi và cách lấy số; §2 Tóm tắt; §3 Tính sẵn sàng; §4 Traffic; §5 Độ trễ; §6 Chat RAG và lỗi 429; §7 Tài nguyên; §8 Thành phần phụ thuộc; §9 Hạn chế của chính metric; §10 Việc cần làm; §11 Cách chạy lại truy vấn; §12 Khung 20:00–21:00; §13 So sánh hai khung và dòng thời gian trong ngày.
- **References**: Prometheus 3.16.0 HTTP API, [`backend_rag/src/common/metrics/prometheus-metrics.ts`](../../backend_rag/src/common/metrics/prometheus-metrics.ts), [`backend_rag/src/modules/rag/transport/client-slots.ts`](../../backend_rag/src/modules/rag/transport/client-slots.ts), [`backend_rag/src/common/security/throttler.ts`](../../backend_rag/src/common/security/throttler.ts), [`backend_rag/src/modules/health/health.service.ts`](../../backend_rag/src/modules/health/health.service.ts), [`backend_rag/docs/PERFORMANCE_AND_CACHING.md`](../../backend_rag/docs/PERFORMANCE_AND_CACHING.md), [`agents/rules/MD_CONVENTION.md`](../../agents/rules/MD_CONVENTION.md).
- **Created**: 2026-11-10T20:15:00+07:00
- **Last Updated**: 2026-11-10T20:40:00+07:00

[STATUS: ACTIVE]

---

## Mục lục

- [1. Phạm vi và cách lấy số](#1-phạm-vi-và-cách-lấy-số)
- [2. Tóm tắt](#2-tóm-tắt)
- [3. Tính sẵn sàng](#3-tính-sẵn-sàng)
- [4. Traffic](#4-traffic)
- [5. Độ trễ theo route](#5-độ-trễ-theo-route)
- [6. Chat RAG và lỗi 429](#6-chat-rag-và-lỗi-429)
- [7. Tài nguyên tiến trình](#7-tài-nguyên-tiến-trình)
- [8. Thành phần phụ thuộc](#8-thành-phần-phụ-thuộc)
- [9. Hạn chế của chính metric](#9-hạn-chế-của-chính-metric)
- [10. Việc cần làm](#10-việc-cần-làm)
- [11. Cách chạy lại truy vấn](#11-cách-chạy-lại-truy-vấn)
- [12. Khung 20:00–21:00](#12-khung-2000-2100)
- [13. So sánh hai khung và dòng thời gian trong ngày](#13-so-sánh-hai-khung-và-dòng-thời-gian-trong-ngày)

---

## 1. Phạm vi và cách lấy số

| Mục | Giá trị |
| :--- | :--- |
| Khung giờ | 2026-11-08 10:00–12:00 `+07:00` (03:00–05:00 UTC) |
| Nguồn | Prometheus 3.16.0, `localhost:9090`, retention 15 ngày |
| Target | `rag_ts_backend` → `host.docker.internal:3000/metrics`, scrape mỗi 5 s, nhãn `service="backend_rag"` |
| Metric dùng | `rag_http_requests_total`, `rag_http_request_duration_seconds_*`, `rag_active_users_{1m,5m}`, `rag_sse_active_streams`, `rag_http_in_flight_requests`, `rag_llm_queue_length`, `rag_component_up`, `rag_uptime_seconds`, `process_*`, `nodejs_*`, `up` |

Cách tính:

- **Số request**: `increase()` bỏ mất mẫu đầu của series mới sinh giữa chừng (ví dụ series `POST /rag/chat/stream 429` chỉ xuất hiện từ 11:06), nên tổng bị lệch khoảng 12 request. Báo cáo dùng giá trị counter thô: series đã có lúc 10:00 thì lấy `cuối − đầu`; series sinh sau 10:00 thì lấy `cuối`, vì counter bắt đầu từ 0 khi tiến trình tạo ra nó. Tiến trình không khởi động lại trong khung giờ (`resets(process_cpu_seconds_total) = 0`), nên cách tính này đúng.
- **Phân vị độ trễ**: nội suy tuyến tính trong bucket histogram `[5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2,5s, 5s, 10s, 30s, +Inf]`. Bucket thưa và mẫu nhỏ, nên chỉ coi phân vị là ước lượng.
- **Gauge**: lấy mẫu mỗi 15 s, nên đỉnh ngắn hơn 15 s có thể bị sót.

---

## 2. Tóm tắt

| # | Phát hiện | Số liệu |
| :--- | :--- | :--- |
| 1 | Backend mất tín hiệu từ **11:34:45** đến hết khung giờ và không tự lên lại | Down 25 phút 15 giây, tức 21% khung giờ. Không có alert. |
| 2 | Gần một nửa yêu cầu chat bị từ chối với **429** | 74/153 lượt, tập trung trong 11:06–11:16 (56% lượt của đoạn này) |
| 3 | Chat chậm: trung bình **26 s** mỗi stream, **23/77** stream trên 30 s | `LLM_CONCURRENCY=1` |
| 4 | API thường (không tính health và chat) đạt p50 22 ms nhưng **p95 ≈ 2,0 s, p99 ≈ 4,2 s** | Đuôi chậm trùng với lúc chat chiếm CPU (đỉnh 338%) |
| 5 | **61%** lượt đọc paper trả về 404; p95 của 404 khoảng 2,1 s | 48/79 lượt |
| 6 | Probe Qdrant báo down 5 lần, mỗi lần ≤ 15 s, đều rơi vào lúc chat | Mongo và LLM luôn up; `world` = 0 suốt khung giờ |
| 7 | Lỗi 5xx rất thấp | 1/318 request không phải health (0,31%) |
| 8 | Metric event loop lag không dùng được để phân tích theo thời gian | Cố định ở 21–23 ms (xem §9) |

---

## 3. Tính sẵn sàng

| Mốc (`+07:00`) | Sự kiện |
| :--- | :--- |
| ≈ 09:38 | Tiến trình khởi động (`rag_uptime_seconds` = 1.337 lúc 10:00) |
| 10:00–10:20 | Up nhưng không có request nào, kể cả health |
| 10:20 | Người dùng đầu tiên vào: health, đăng ký, chat đầu tiên |
| 11:06–11:17 | Đợt tải chính |
| 11:32:00 | Mẫu `up = 1` cuối cùng còn ổn định |
| **11:34:45** | `up` chuyển sang 0 và giữ 0 đến 12:00 |

Ngay trước lúc mất tín hiệu, mọi gauge đều bình thường: CPU 0,1–36%, RSS 62 MB, heap 128 MB, không có stream hay hàng đợi LLM nào đang chạy. Như vậy không có dấu hiệu quá tải hay hết bộ nhớ. Metric không cho biết nguyên do: có thể tiến trình bị dừng tay, crash do lỗi chưa bắt, hoặc máy chủ mất mạng tới container Prometheus. Cần đối chiếu với log của tiến trình.

Uptime trong khung giờ: **94 phút 45 giây / 120 phút = 79%**.

---

## 4. Traffic

### 4.1 Tổng theo route

| Route | Status | Số lượt |
| :--- | :--- | ---: |
| `GET /api/v1/health` | 200 | 385 |
| `GET /api/v1/health` | 304 | 75 |
| `POST /api/v1/rag/chat/stream` | 200 | 77 |
| `POST /api/v1/rag/chat/stream` | 429 | 74 |
| `POST /api/v1/rag/chat/stream` | 401 | 2 |
| `GET /api/v1/papers/:paper_id` | 404 | 48 |
| `GET /api/v1/papers/:paper_id` | 200 | 30 |
| `GET /api/v1/papers/:paper_id` | 503 | 1 |
| `PATCH /api/v1/chat-sessions/:id` | 200 | 21 |
| `GET /api/v1/chat-sessions` | 304 | 19 |
| `GET /api/v1/chat-sessions` | 200 | 15 |
| `GET /api/v1/chat-sessions` | 401 | 4 |
| `GET /api/v1/chat-sessions/:id/messages` | 404 | 6 |
| `POST /api/v1/users` | 201 / 400 / 409 | 8 / 5 / 1 |
| `POST /api/v1/auth/token-rotations` | 201 | 4 |
| `POST /api/v1/auth/tokens` | 201 | 1 |
| `POST /api/v1/auth/token-revocations` | 204 | 1 |
| `DELETE /api/v1/chat-sessions/:id` | 204 | 1 |
| **Tổng** | | **778** |

- Health probe chiếm **460/778 (59%)**. Đây là frontend tự poll, không phải người dùng thao tác.
- Trong 318 request còn lại: 2xx 158 (49,7%), 3xx 19 (6,0%), 4xx 140 (44,0%), 5xx 1 (0,3%). Nhóm 4xx chủ yếu gồm 429 của chat (74) và 404 của paper (48).
- Đỉnh người dùng hoạt động: **9** trong 1 phút (11:15:15) và **11** trong 5 phút.
- Khoảng 8 tài khoản mới đăng ký thành công. Có 5 lượt đăng ký lỗi validate (400) và 1 lượt trùng email (409).

### 4.2 Theo thời gian (mỗi 10 phút)

| Khung | Health | Chat 200 | Chat 429 | Request khác | CPU TB / đỉnh (%) | RSS đỉnh (MB) | User 1m đỉnh | SSE đỉnh |
| :--- | ---: | ---: | ---: | ---: | :--- | ---: | ---: | ---: |
| 10:00 | 0 | 0 | 0 | 0 | 0,6 / 0,8 | 304 | 0 | 0 |
| 10:10 | 0 | 0 | 0 | 0 | 0,6 / 0,8 | 152 | 0 | 0 |
| 10:20 | 29 | 2 | 0 | 6 | 10 / 84 | 2.276 | 2 | 1 |
| 10:30 | 44 | 0 | 0 | 1 | 0,6 / 0,7 | 91 | 1 | 0 |
| 10:40 | 41 | 1 | 0 | 9 | 4 / 44 | 167 | 2 | 1 |
| 10:50 | 82 | 5 | 0 | 25 | 24 / 146 | 2.250 | 2 | 1 |
| **11:00** | 104 | 28 | **31** | 45 | **124 / 338** | 2.314 | 6 | 3 |
| **11:10** | 101 | 34 | **43** | 48 | **157 / 302** | 2.178 | **9** | 3 |
| 11:20 | 32 | 6 | 0 | 14 | 29 / 123 | 2.304 | 2 | 2 |
| 11:30 | 27 | 1 | 0 | 19 | 6,5 / 36 | 324 | 4 | 1 |
| 11:40–12:00 | không có dữ liệu (backend down) | | | | | | | |

Toàn bộ 429 rơi vào 11:06–11:16, đúng lúc nhiều người chat cùng lúc.

---

## 5. Độ trễ theo route

Phân vị ước lượng từ bucket, đơn vị ms. Cột "n" là số lượt.

| Route · status | n | Trung bình | p50 | p95 | p99 |
| :--- | ---: | ---: | ---: | ---: | ---: |
| `GET /health` 200 | 385 | 2,4 | 2,8 | 8,8 | 19,7 |
| `GET /chat-sessions` 200 | 15 | 13,6 | 11,5 | 43,8 | 48,8 |
| `GET /chat-sessions` 304 | 19 | 21,2 | 3,7 | 262,5 | 452,5 |
| `PATCH /chat-sessions/:id` 200 | 21 | 84,8 | 21,2 | 487,5 | 895,0 |
| `GET /papers/:paper_id` 200 | 30 | 637,0 | 193,8 | 3.125 | 4.625 |
| `GET /papers/:paper_id` 404 | 48 | 288,2 | 20,4 | 2.080 | 3.800 |
| `POST /users` 201 | 8 | 584,1 | 83,3 | 4.000 | 4.800 |
| `POST /auth/token-rotations` 201 | 4 | 361,0 | 50,0 | 2.200 | 2.440 |
| `POST /auth/tokens` 201 | 1 | 2.002 | — | — | — |
| `DELETE /chat-sessions/:id` 204 | 1 | 3.369 | — | — | — |
| `POST /rag/chat/stream` 429 | 74 | 181,3 | 9,4 | 935 | 1.945 |
| `POST /rag/chat/stream` 200 | 77 | **25.958** | 22.439 | > 30.000 | > 30.000 |

Gộp mọi route trừ health và chat stream (n = 165): **p50 21,6 ms · p95 ≈ 2.010 ms · p99 ≈ 4.175 ms · trung bình 292 ms**.

Nhận xét:

- Lúc rảnh, phần lớn route trả về dưới 50 ms. Đuôi chậm (giây) xuất hiện cùng đợt chat 11:00–11:20, khi CPU tiến trình lên 300%. Như vậy Laya, retrieval và các request thường dùng chung CPU và event loop của cùng một tiến trình Node, và request thường chịu chậm theo.
- Nếu so với ngưỡng SLA trong benchmark (`P95 < 500 ms`, `P99 < 1.000 ms`, D-019), buổi này **không đạt** ở p95/p99. Lưu ý: SLA đó áp cho bài k6 có kiểm soát; ở đây mẫu chỉ có 165 lượt và đang trộn với tải chat, nên đây là tín hiệu cần đo lại chứ chưa phải kết luận.
- Đăng ký 201 có p50 83 ms nhưng p95 khoảng 4 s. Argon2id (64 MB, t=3, p=4, D-021) cạnh tranh threadpool và CPU với Laya. Đây là chi phí dự kiến; không được giảm tham số Argon2id để xử lý.
- 404 của paper có p95 khoảng 2,1 s: không tìm thấy vẫn tốn một vòng tra cứu đầy đủ, vì kết quả âm không được cache.

---

## 6. Chat RAG và lỗi 429

### 6.1 Phân bố thời gian stream (77 lượt 200)

| Khoảng | Số lượt | Tỉ lệ |
| :--- | ---: | ---: |
| ≤ 2,5 s | 1 | 1% |
| 2,5–5 s | 6 | 8% |
| 5–10 s | 6 | 8% |
| 10–30 s | 41 | 53% |
| **> 30 s** | **23** | **30%** |

Histogram dừng ở bucket 30 s, nên không biết đuôi dài tới đâu (`LLM_TIMEOUT_MS` = 60 s, `LAYA_TIMEOUT_MS` = 30 s). Metric đo toàn bộ thời gian stream SSE, không tách thời gian tới token đầu tiên (TTFT), nên không biết người dùng phải chờ bao lâu mới thấy chữ đầu.

### 6.2 Lỗi 429 từ đâu

Có ba chỗ trong code có thể trả 429 cho `POST /rag/chat/stream`:

| Nguồn | Ngưỡng | Số liệu Prometheus | Kết luận |
| :--- | :--- | :--- | :--- |
| Throttler `ragChat` | 20 lượt / 60 s mỗi IP (10 nếu không có token) | Đỉnh **22 lượt** trong một cửa sổ 60 s trượt (quanh 11:08); trong một phút lịch, đỉnh là 18 | Có thể gây ra **một phần** 429, chỉ khi mọi người dùng chung một IP |
| Hàng đợi LLM đầy | `LLM_MAX_QUEUE=8`, 1 slot | `rag_llm_queue_length` đỉnh 3 | Không phải |
| `ClientSlots` | 3 request đang chạy mỗi IP, 2 mỗi user | `rag_sse_active_streams` đỉnh đúng **3** | **Khả năng cao nhất** cho phần lớn 429 |

Giả thuyết: người dùng đi qua cùng một địa chỉ (NAT, proxy hoặc tunnel), trong khi `TRUST_PROXY_HOPS=0`. Khi đó `req.ip` của mọi người giống nhau. Cả nhóm chỉ có 3 slot chat, nên người thứ tư bị trả `too many concurrent requests` ngay lập tức (p50 9 ms). Cả nhóm cũng chỉ có chung ngân sách throttler 20 lượt/phút, và ngân sách này bị vượt nhẹ quanh 11:08. Cả hai cơ chế đều cần giả thuyết chung IP mới giải thích được 74 lượt 429 từ 9–11 người dùng. Metric không ghi `req.ip` hay message của lỗi, nên cần xác nhận bằng log sự kiện `CHAT_TOO_MANY_INFLIGHT` và message `ThrottlerException`.

Hệ quả: với `LLM_CONCURRENCY=1` và khoảng 26 s mỗi câu, hệ thống trả lời được tối đa khoảng 2,3 câu/phút. Trong đợt tải, người dùng gửi tới 18 lượt/phút. Kể cả khi bỏ giới hạn slot, phần lớn yêu cầu vẫn phải xếp hàng lâu. 429 chỉ chuyển việc chờ thành từ chối sớm.

---

## 7. Tài nguyên tiến trình

| Metric | Rảnh | Khi chat | Đỉnh |
| :--- | :--- | :--- | :--- |
| CPU (`rate(process_cpu_seconds_total[1m])`, 100% = 1 core) | 0,5–0,7% | 120–300% | **338%** (11:08) |
| RSS | 60–250 MB | 1,5–2,3 GB | **2.314 MB** (11:02:30) |
| Heap used | 90–97 MB | 115–133 MB | 132,9 MB (11:24) |
| In-flight request | 0 | 2–4 | 4 |

- RSS nhảy lên khoảng 1,6–2,2 GB mỗi khi có chat, rồi rơi về dưới 100 MB khi rảnh. Mức tăng khớp với bộ nhớ của Laya ONNX (khoảng 1,6 GB theo đo trước đây). Bộ nhớ ngoài heap được trả lại hệ điều hành, không có dấu hiệu rò.
- Heap tăng từ 91 MB (10:00) lên 128–133 MB (11:20–11:34), sau đó đi ngang. Hai giờ quá ngắn để kết luận về rò bộ nhớ; nên theo dõi qua một phiên dài hơn.
- CPU vượt 100% nghĩa là có nhiều luồng chạy song song (ONNX runtime, threadpool của Argon2id), không chỉ event loop.

---

## 8. Thành phần phụ thuộc

`rag_component_up` (1 = up), lấy mẫu mỗi 15 s:

| Thành phần | Trạng thái |
| :--- | :--- |
| `mongo` | Up suốt khung giờ |
| `llm` | Up suốt khung giờ |
| `qdrant` | Down 5 lần, mỗi lần một mẫu (≤ 15 s): 10:21:45, 11:06:30, 11:09:15, 11:11:45, 11:13:45 |
| `world` | **0 suốt khung giờ**: tree-router chưa nạp world |

- Cả 5 lần Qdrant "down" đều trùng lúc có chat. Lần đầu (10:21:45) rơi đúng vào câu chat đầu tiên của buổi. Có thể ping Qdrant timeout khi CPU tiến trình bị Laya chiếm, chứ không phải Qdrant thật sự sập: trong ±1 phút quanh mỗi lần, `GET /papers/:paper_id` vẫn trả 200 (1–5 lượt mỗi lần), và chỉ có đúng 1 lượt 503 (`GET /papers/:paper_id` lúc 11:08). Theo `overallStatus()`, khi `qdrant=false` thì `/health` trả trạng thái `down`, nên frontend có thể hiển thị sai trạng thái trong những khoảnh khắc này.
- `world=0` nghĩa là router không nạp được file world. Cần kiểm tra xem truy xuất có đang chạy ở chế độ dự phòng (không dùng tree-router) hay không.

---

## 9. Hạn chế của chính metric

| Vấn đề | Chi tiết | Ảnh hưởng |
| :--- | :--- | :--- |
| Event loop lag là số cộng dồn | `monitorEventLoopDelay({ resolution: 20 })` không bao giờ `reset()`. `mean` và `p99` tính từ lúc tiến trình khởi động, và sàn bằng độ phân giải 20 ms. | Giá trị đứng yên 21–23 ms suốt buổi, kể cả lúc CPU 338%. Không phát hiện được event loop bị chặn. |
| Histogram dừng ở 30 s | Bucket cuối trước `+Inf` là 30 s | 30% stream chat không đo được độ dài |
| Không có TTFT | Chỉ có tổng thời gian stream | Không đo được trải nghiệm chờ chữ đầu tiên |
| 429 không có nhãn lý do | Throttler, hàng đợi và slot đều ra `status="429"` | Phải đoán nguồn (§6.2) |
| `process_open_fds` không có trong series của backend | Truy vấn với `job="rag_ts_backend"` trả về rỗng | Không theo dõi được rò file descriptor |
| Không có rule cảnh báo | Backend down 25 phút mà không ai được báo | Mất tín hiệu chỉ thấy khi xem lại |

---

## 10. Việc cần làm

| Ưu tiên | Việc | Lý do (mục) |
| :--- | :--- | :--- |
| P0 | Tra log quanh 11:34:45 ngày 08/11 để tìm nguyên nhân backend dừng; thêm alert `up{job="rag_ts_backend"} == 0` kéo dài 1 phút | §3 |
| P0 | Xác nhận nguồn 429 qua log `CHAT_TOO_MANY_INFLIGHT` và `ThrottlerException`. Nếu người dùng thật sự đi qua proxy hoặc tunnel, đặt `TRUST_PROXY_HOPS` bằng đúng số proxy (D-054). Không nới `MAX_INFLIGHT_PER_IP` một cách tùy tiện. | §6.2 |
| P1 | Thêm nhãn `reason` (`throttle` / `queue_full` / `inflight`) cho 429; thêm bucket 60 s và 120 s; thêm histogram TTFT cho chat | §6, §9 |
| P1 | Sửa metric event loop: `reset()` sau mỗi lần scrape hoặc dùng cửa sổ trượt; giảm `resolution` xuống 10 ms | §9 |
| P1 | Probe Qdrant: chỉ báo down sau 2 lần lỗi liên tiếp, hoặc tăng timeout của ping | §8 |
| P1 | Kiểm tra vì sao `world` = 0 | §8 |
| P1 | Tìm nguồn hơn 60% lượt paper trả 404 (lặp lại ở cả hai khung, xem §13) (id từ frontend không có trong corpus, hoặc chunk chưa đồng bộ); cache kết quả âm trong thời gian ngắn | §4, §5 |
| P2 | Tách Laya khỏi event loop chính (worker thread hoặc tiến trình riêng) để chat không kéo theo p95 của API thường | §5, §7 |
| P2 | Giãn chu kỳ poll `/health` của frontend, vì health hiện chiếm 59% request | §4 |

---

## 11. Cách chạy lại truy vấn

Mọi số trong báo cáo lấy từ HTTP API của Prometheus, ví dụ:

```bash
P=http://localhost:9090/api/v1
S=2026-11-08T03:00:00Z; E=2026-11-08T05:00:00Z   # 10:00–12:00 +07:00

# Counter thô theo route/status (bước 15 s) để tính số lượt đúng
curl -sG "$P/query_range" --data-urlencode 'query=rag_http_requests_total' \
  --data-urlencode start=$S --data-urlencode end=$E --data-urlencode step=15

# Bucket histogram để ước lượng phân vị
curl -sG "$P/query_range" --data-urlencode 'query=rag_http_request_duration_seconds_bucket' \
  --data-urlencode start=$S --data-urlencode end=$E --data-urlencode step=15

# CPU, RSS, gauge của RAG
curl -sG "$P/query_range" --data-urlencode 'query=rate(process_cpu_seconds_total{job="rag_ts_backend"}[1m])*100' \
  --data-urlencode start=$S --data-urlencode end=$E --data-urlencode step=15
curl -sG "$P/query_range" --data-urlencode 'query=rag_component_up' \
  --data-urlencode start=$S --data-urlencode end=$E --data-urlencode step=15
```

Dữ liệu còn trong Prometheus tới khoảng ngày 23/11/2026 (retention 15 ngày). Nếu cần giữ lâu hơn, hãy xuất ra file trước mốc đó.

Với khung 20:00–21:00, dùng `S=2026-11-08T13:00:00Z; E=2026-11-08T14:00:00Z`. Khung này có một lần khởi động lại, nên khi cộng counter phải coi mẫu đầu tiên sau mốc 20:20:44 là đếm lại từ 0. `increase()` của Prometheus đã tự xử lý việc này, nhưng vẫn bỏ sót mẫu đầu của các series mới sinh.

---

## 12. Khung 20:00–21:00

Cách lấy số giống §1, với bước 5 s. Tiến trình đang chạy lúc 20:00 được khởi động lúc **19:57:55**.

### 12.1 Tính sẵn sàng

| Mốc (`+07:00`) | Sự kiện |
| :--- | :--- |
| 19:57:55 | Tiến trình khởi động (`rag_uptime_seconds` = 124 lúc 20:00) |
| 20:03 | Request đầu tiên |
| **20:20:44** | **Khởi động lại**: `up` mất 2 mẫu scrape (20:20:40–20:20:50), uptime về 0, counter reset; `rag_component_up{component="llm"}` = 0 trong 10 s đầu (20:20:50–20:21:00) trong lúc nạp model |
| 20:21–21:00 | Chạy ổn định tới hết khung giờ |

Lần khởi động lại xảy ra lúc hệ thống rảnh: trong 2 phút trước đó CPU 0,5–0,9%, RSS 72–89 MB, không có stream hay hàng đợi LLM nào đang chạy. Như vậy đây không phải crash do tải. Nhiều khả năng là khởi động lại có chủ đích (deploy, `--watch`, hoặc chạy lại tay); cần đối chiếu log để chắc chắn. Downtime khoảng 10 s, uptime của khung **≈ 99,7%**.

### 12.2 Traffic

| Route | Status | Số lượt |
| :--- | :--- | ---: |
| `GET /api/v1/health` | 200 / 304 | 149 / 46 |
| `GET /api/v1/papers/:paper_id` | 404 / 200 / 304 | 25 / 10 / 3 |
| `POST /api/v1/rag/chat/stream` | 200 / 401 | 24 / 1 |
| `GET /api/v1/chat-sessions` | 304 / 401 / 200 | 22 / 7 / 4 |
| `PATCH /api/v1/chat-sessions/:id` | 200 | 18 |
| `POST /api/v1/auth/token-rotations` | 201 / 401 | 5 / 1 |
| `GET /api/v1/chat-sessions/:id/messages` | 404 / 200 | 2 / 1 |
| `POST /api/v1/users` | 201 / 400 | 1 / 1 |
| **Tổng** | | **320** |

- Health chiếm **195/320 (61%)**, cùng tỉ lệ với buổi sáng.
- Trong 125 request còn lại: 2xx 63 (50%), 3xx 25 (20%), 4xx 37 (30%), **5xx 0**.
- **Không có 429 nào.** Chat rải đều, tối đa 3 lượt/phút (20:05, 20:55); SSE đồng thời tối đa 2; hàng đợi LLM tối đa 1.
- Người dùng hoạt động: tối đa **3** trong 1 phút, **5** trong 5 phút.
- 404 của paper vẫn chiếm **25/38 (66%)** lượt đọc paper.
- Có 7 lượt `GET /chat-sessions` 401 và 1 lượt làm mới token 401 (mất 2,19 s). Phiên đăng nhập hết hạn được frontend xử lý bằng cách gọi lại; đây không phải lỗi máy chủ.

Theo thời gian (mỗi 10 phút):

| Khung | Health | Chat 200 | Request khác | CPU TB / đỉnh (%) | RSS đỉnh (MB) | User 1m đỉnh | SSE đỉnh |
| :--- | ---: | ---: | ---: | :--- | ---: | ---: | ---: |
| 20:00 | 24 | 9 | 26 | 36 / 162 | 1.790 | 3 | 2 |
| 20:10 | 60 | 5 | 27 | 24 / 101 | 2.329 | 3 | 1 |
| 20:20 | 42 | 4 | 12 | 20 / 67 | 2.280 | 3 | 1 |
| 20:30 | 21 | 0 | 8 | 0,6 / 0,8 | 161 | 3 | 0 |
| 20:40 | 14 | 1 | 3 | 6 / 54 | 2.131 | 3 | 1 |
| 20:50 | 34 | 5 | 25 | 21 / 82 | 1.740 | 2 | 1 |

### 12.3 Độ trễ

| Route · status | n | Trung bình | p50 | p95 | p99 |
| :--- | ---: | ---: | ---: | ---: | ---: |
| `GET /health` 200 | 149 | 2,1 | 2,6 | 5,4 | 9,6 |
| `GET /chat-sessions` 304 | 22 | 6,0 | 3,9 | 24,5 | 44,5 |
| `PATCH /chat-sessions/:id` 200 | 18 | 25,4 | 22,3 | 55,0 | 91,0 |
| `GET /papers/:paper_id` 200 | 10 | 111,5 | 83,3 | 225,0 | 245,0 |
| `GET /papers/:paper_id` 404 | 25 | 249,9 | 21,8 | 2.031 | 2.406 |
| `POST /auth/token-rotations` 201 | 5 | 64,0 | 43,8 | 212,5 | 242,5 |
| `POST /rag/chat/stream` 200 | 24 | **27.140** | 22.632 | > 30.000 | > 30.000 |

Gộp mọi route trừ health và chat (n = 100): **p50 19,2 ms · p95 ≈ 500 ms · p99 ≈ 2.200 ms · trung bình 119 ms**.

- Khi tải nhẹ, các route phiên chat và paper 200 đều nằm dưới 250 ms ở p95. Đuôi p99 khoảng 2,2 s lúc này gần như chỉ do **404 của paper** (p95 khoảng 2,0 s) và một lượt làm mới token 401. Vấn đề 404 chậm (§5) xảy ra ngay cả khi không có tải chat, nên đây là lỗi riêng của đường xử lý 404, không phải do tranh CPU.
- Chat vẫn chậm dù gần như không phải xếp hàng (hàng đợi LLM ≤ 1): **cả 24/24** stream đều trên 10 s, **5/24 (21%)** trên 30 s. Như vậy độ trễ chat chủ yếu là thời gian xử lý của chính một câu (Laya + LLM 7B Q4), không phải thời gian chờ. Tăng giới hạn đồng thời sẽ không làm câu đơn lẻ nhanh hơn.

### 12.4 Tài nguyên và thành phần

| Metric | Giá trị |
| :--- | :--- |
| CPU | Trung bình 17,8%, đỉnh 162% (20:05:45), lúc rảnh 0,4–0,8% |
| RSS | 42 MB – **2.329 MB** (20:14:15). Vẫn là kiểu nhảy lên khoảng 1,6–2,3 GB mỗi khi có chat rồi rơi về dưới 100 MB |
| Heap used | 86–129 MB; sau khi khởi động lại bắt đầu từ 86 MB và lên 120 MB lúc 21:00 |
| Event loop lag | 21–25 ms, p99 có một lần 33 ms ngay sau khởi động lại (20:22). Đây là số cộng dồn từ lúc khởi động (xem §9), nên lần nhảy này chỉ phản ánh lúc tiến trình mới chạy, khi mẫu còn ít |
| `mongo` | Up suốt khung giờ |
| `llm` | Down 10 s ngay sau khởi động lại, rồi up |
| `qdrant` | Down **1 lần**, khoảng 5 s (20:22:35), ngay khi chat đầu tiên sau khởi động lại chạy Laya. Cùng kiểu với 5 lần buổi sáng |
| `world` | **0 suốt khung giờ**, giống buổi sáng |

---

## 13. So sánh hai khung và dòng thời gian trong ngày

| Chỉ số | 10:00–12:00 | 20:00–21:00 |
| :--- | :--- | :--- |
| Thời lượng có dữ liệu | 94 phút 45 giây / 120 phút | ≈ 59 phút 50 giây / 60 phút |
| Sự cố tiến trình | Dừng 11:34:45, không lên lại trong khung | Khởi động lại 20:20:44 lúc rảnh, mất khoảng 10 s |
| Tổng request / không tính health | 778 / 318 | 320 / 125 |
| Người dùng 1 phút / 5 phút (đỉnh) | 9 / 11 | 3 / 5 |
| Lượt chat (200 / 429 / 401) | 77 / **74** / 2 | 24 / **0** / 1 |
| Chat trung bình · tỉ lệ > 30 s | 26,0 s · 30% | 27,1 s · 21% |
| SSE đồng thời / hàng đợi LLM (đỉnh) | 3 / 3 | 2 / 1 |
| API không tính health và chat: p50 / p95 / p99 | 21,6 ms / 2.010 ms / 4.175 ms | 19,2 ms / 500 ms / 2.200 ms |
| Paper 404 · p95 của 404 | 61% · 2,08 s | 66% · 2,03 s |
| 5xx | 1 (0,3%) | 0 |
| CPU đỉnh · RSS đỉnh | 338% · 2.314 MB | 162% · 2.329 MB |
| Qdrant probe báo down | 5 lần | 1 lần |
| `world` | 0 | 0 |

Kết luận rút ra từ hai khung:

1. **429 chỉ xuất hiện khi tải cao.** Ở mức 3 người dùng không có lượt nào; ở mức 9–11 người dùng thì gần một nửa lượt bị từ chối. Điều này khớp với giả thuyết giới hạn theo IP ở §6.2: mỗi IP chỉ có 3 slot và 20 lượt/phút.
2. **Độ trễ của một câu chat không phụ thuộc tải**: khoảng 26–27 s ở cả hai khung. Muốn câu trả lời nhanh hơn phải giảm thời gian xử lý từng câu (LLM, Laya), không phải nới giới hạn đồng thời.
3. **p95 của API thường phụ thuộc tải**: khoảng 0,5 s lúc tải nhẹ và khoảng 2,0 s lúc tải cao. Điều này củng cố đề xuất tách Laya khỏi event loop (§10).
4. **Có hai vấn đề cố định ở cả hai khung**: 404 của paper (hơn 60%, chậm khoảng 2 s) và `world=0`. Hai việc này nên ưu tiên lên P1.
5. **Probe Qdrant báo down mỗi khi Laya bắt đầu chạy**, kể cả lúc tải nhẹ, nên khả năng cao là timeout của probe chứ không phải Qdrant hỏng.

Dòng thời gian tiến trình trong ngày (từ `rag_uptime_seconds` và `up`):

| Khởi động | Dừng | Ghi chú |
| :--- | :--- | :--- |
| 09:37:43 | 11:34:45 | Khung sáng; dừng không rõ nguyên nhân (§3) |
| 15:47:39 | 17:29:30 | Ngoài phạm vi báo cáo |
| 19:57:55 | 20:20:44 | Khởi động lại lúc rảnh |
| 20:20:44 | sau 21:00 | Chạy ổn định tới hết khung tối |

Trong ngày có hai đoạn backend down dài (11:34–15:47 và 17:29–19:58) mà không có cảnh báo, nên việc thêm alert `up == 0` (P0 ở §10) càng cần thiết.
