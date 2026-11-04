# API-compare.md — So sánh API trước và sau khi thiết kế lại theo 8 nguyên tắc

- **Motivation/Background**: API cũ lệch giữa FE, BE và tài liệu, URL còn chứa động từ, và chưa có XML, HATEOAS hay idempotency. API đã được thiết kế lại theo 8 nguyên tắc RESTful của dự án, nên cần một bản ghi cho thấy cái gì đã sai và cái gì đã được sửa.
- **Purpose**: Liệt kê các lỗi của thiết kế cũ, các cải thiện của thiết kế hiện tại và toàn bộ API đang có.
- **Overview Pipeline**: Bản cũ đọc từ commit `987a97d~1` của `backend_rag`. Bản mới đọc từ các controller hiện tại, rồi đối chiếu với test `API contract (OpenAPI)`.
- **Detailed Plan**: §1 Tóm tắt; §2 Lỗi của thiết kế cũ; §3 Nội dung 8 nguyên tắc, đánh giá và cải thiện; §4 Toàn bộ API hiện tại; §5 Bảng đổi route; §6 Hạn chế và việc còn mở.
- **References**: [docs/API_DESIGN.md](docs/API_DESIGN.md), [docs/API_SPECIFICATION.md](docs/API_SPECIFICATION.md), [docs/AI_chat_log/Vuong_chat_logs.md](docs/AI_chat_log/Vuong_chat_logs.md) (D-047 đến D-053), [backend_rag/test/flow.e2e-spec.ts](backend_rag/test/flow.e2e-spec.ts), [backend_rag/src/common/pagination.ts](backend_rag/src/common/pagination.ts), [backend_rag/src/common/xml.ts](backend_rag/src/common/xml.ts), [backend_rag/src/common/hateoas.ts](backend_rag/src/common/hateoas.ts).
- **Created**: 2026-11-04T23:58:56+07:00
- **Last Updated**: 2026-11-05T00:01:07+07:00

[STATUS: ACTIVE]

---

## 1. Tóm tắt

Thiết kế API đã xong và đã khớp giữa BE và FE, nhưng **`POST /rag/chat*` và `GET /papers*` chưa trả được nội dung thật**. Lý do là pipeline đã bỏ `text` khỏi payload Qdrant, nên BE phải đọc lại từ `rag_ts_corpus` (§6).

| Chỉ số | Thiết kế cũ | Hiện tại |
| :--- | :--- | :--- |
| Tổng số route | 30 | 34 |
| Route có động từ trong URL | 8 | 0 |
| Định dạng dữ liệu | JSON | JSON và XML |
| Route có `_links` | 0 | 26/34 (§3, nguyên tắc 8) |
| Phân trang và lọc | Mỗi route một kiểu | Dùng chung `limit`, `cursor`, `sort`, filter |
| Chống chạy lặp chat | Không có | `Idempotency-Key` |
| Trả 405 khi sai method | Không | Có, kèm header `Allow` |

Trong 30 route cũ, 17 route giữ nguyên path và 13 route đổi thành 12 route mới. Có thêm 5 route mới hoàn toàn. Không giữ alias cho route cũ, vì FE là client duy nhất và được phát hành cùng BE ([D-047](docs/AI_chat_log/Vuong_chat_logs.md)).

---

## 2. Lỗi của thiết kế cũ

### 2.1 URL và HTTP method

| # | Lỗi | Ví dụ ở bản cũ | Hậu quả |
| :--- | :--- | :--- | :--- |
| L1 | Dùng động từ trong URL | `/auth/login`, `/auth/register`, `/auth/refresh`, `/auth/logout`, `/auth/change-password`, `/admin/users/:id/disable`, `/enable`, `/revoke-tokens` | URL mô tả hành động thay vì tài nguyên, và trái nguyên tắc 1. |
| L2 | Gộp tài khoản vào nhóm auth | `GET /auth/me`, `PATCH /auth/me` | Hồ sơ người dùng và vòng đời token nằm chung một nhóm. |
| L3 | Đặt tên không thống nhất | `/chat-sessions` ở phía user nhưng `/admin/sessions` và `/admin/users/:id/sessions` ở phía admin. Query dùng `active` nhưng field trả về là `is_active`. | Client phải nhớ các trường hợp ngoại lệ. |
| L4 | Chọn sai method | `POST` cho disable, enable, revoke-tokens. `PATCH` cho roles dù body luôn là toàn bộ roles. | Không suy ra được route có an toàn khi gọi lặp hay không. |

### 2.2 Status code

| # | Lỗi | Ví dụ ở bản cũ | Hậu quả |
| :--- | :--- | :--- | :--- |
| L5 | Cùng là tạo token nhưng trả status khác nhau | `register` trả 201, còn `login` và `refresh` trả 200, dù đều trả `AuthResponseDto` | Không nhất quán giữa các route cùng loại. |
| L6 | Không phân biệt path sai và method sai | Không có xử lý 405 và header `Allow` | Client không biết path có tồn tại hay không. |
| L7 | Không có 406 và 415 | Không có kiểm tra `Accept` và `Content-Type` ở tầng API | Client không nhận được lỗi định dạng rõ ràng. |

### 2.3 Lọc và phân trang

| # | Lỗi | Ví dụ ở bản cũ | Hậu quả |
| :--- | :--- | :--- | :--- |
| L8 | FE và BE lệch contract | FE gọi `GET /chat-sessions?limit=50`, nhưng DTO của BE không có `limit` | **Lỗi thật:** bị trả 400 và không tải được lịch sử chat. |
| L9 | Cursor admin hỏng từ trang 2 | Validator đòi `id` là UUID, trong khi `users._id` là `ObjectId` | **Lỗi thật:** cursor do chính server tạo ra lại bị từ chối. |
| L10 | Trang và sort bị cố định | `/chat-sessions` luôn 10 dòng và luôn sort `last_message_at` | Client không chọn được kích thước trang hay thứ tự. |
| L11 | Filter không có ngôn ngữ chung | Mỗi route tự có tham số riêng (`role`, `active`, `actor_id`, `action`), không có toán tử so sánh | Không lọc được theo khoảng thời gian. |
| L12 | Danh sách không cùng cấu trúc | `/admin/login-attempts` chỉ trả `items`, không có cursor và `has_more` | Không phân trang được, client phải xử lý riêng. |
| L13 | Tin nhắn không phân trang được | `GET /chat-sessions/:id` chỉ trả tối đa 200 tin mới nhất | Không đọc được tin cũ hơn trong phiên dài. |
| L14 | Không có thao tác theo lô | Phải gọi từng `GET /papers/:id` và từng `DELETE /chat-sessions/:id` | Một màn hình phát sinh nhiều request. |

### 2.4 Idempotency

| # | Lỗi | Ví dụ ở bản cũ | Hậu quả |
| :--- | :--- | :--- | :--- |
| L15 | Thử lại chat bị chạy lặp | `POST /rag/chat*` không có khoá chống lặp | LLM chạy lại và lưu trùng tin nhắn hoặc trùng phiên. |
| L16 | Gọi lặp vẫn gây tác dụng phụ | `PATCH roles` với roles không đổi vẫn tăng `token_version`, thu hồi toàn bộ token và ghi audit | Người dùng bị đăng xuất dù quyền không đổi, và audit bị nhiễu. |

### 2.5 Định dạng, hypermedia và tài liệu

| # | Lỗi | Ví dụ ở bản cũ | Hậu quả |
| :--- | :--- | :--- | :--- |
| L17 | Chỉ hỗ trợ JSON | Không có xử lý XML | Không đạt nguyên tắc 4. |
| L18 | Không có HATEOAS | Không có `_links` và không có điểm vào `GET /api/v1` | Client phải hard-code mọi đường dẫn. |
| L19 | FE không xoay vòng token | [`authStore`](frontend_rag/src/store/authStore.ts) cũ chỉ gọi `login`, `register`, `logout` | BE có refresh token nhưng store này không dùng. |
| L20 | Test contract còn hẹp | Test OpenAPI cũ chỉ liệt kê 17 route, không có 13 route admin, và không kiểm tra cách đặt tên | Admin và quy tắc đặt tên có thể lệch mà test không phát hiện. |

---

## 3. Cải thiện theo 8 nguyên tắc

### 3.1 Nội dung 8 nguyên tắc

Đây là nguyên văn 8 nguyên tắc mà dự án dùng làm chuẩn.

1. **Sử dụng danh từ cho URL:** URL nên sử dụng danh từ để đại diện cho tài nguyên, thay vì sử dụng động từ. Điều này giúp URL trở nên trực quan và dễ hiểu. Ví dụ: `/users` thay vì `/getUsers`.
2. **Sử dụng chuẩn HTTP status codes:** Sử dụng đúng mã trạng thái HTTP để phản hồi. Một số mã trạng thái thường dùng bao gồm:
   - `200 OK`: Yêu cầu thành công.
   - `201 Created`: Tạo mới thành công.
   - `204 No Content`: Xóa thành công nhưng không có dữ liệu trả về.
   - `400 Bad Request`: Yêu cầu không hợp lệ.
   - `401 Unauthorized`: Yêu cầu cần xác thực.
   - `403 Forbidden`: Bị cấm truy cập tài nguyên.
   - `404 Not Found`: Không tìm thấy tài nguyên.
   - `500 Internal Server Error`: Lỗi từ server.
3. **Cung cấp API versioning:** Để duy trì tính ổn định và tương thích ngược, cung cấp phiên bản cho API là cần thiết. Có thể thực hiện bằng cách thêm phiên bản vào URL (ví dụ: `/v1/users`) hoặc sử dụng header `Accept` (ví dụ: `Accept: application/vnd.yourapi.v1+json`).
4. **Hỗ trợ định dạng dữ liệu nhiều loại:** Đảm bảo API hỗ trợ nhiều định dạng dữ liệu, chẳng hạn như JSON và XML. Sử dụng header `Content-Type` để xác định định dạng của dữ liệu gửi và nhận.
5. **Sử dụng các bộ lọc và phân trang:** Để giảm tải dữ liệu và cải thiện hiệu suất, nên cung cấp các bộ lọc và phân trang. Ví dụ: `/users?page=2&size=10` để lấy trang thứ hai của danh sách người dùng, mỗi trang có 10 người dùng.
6. **Sử dụng OAuth hoặc token-based authentication:** Để bảo mật, sử dụng phương thức xác thực như OAuth hoặc token-based authentication thay vì thông tin đăng nhập thông thường. Điều này giúp bảo vệ tài nguyên và dữ liệu người dùng.
7. **Đặt tên rõ ràng và thống nhất:** Đặt tên các endpoint và tài nguyên một cách rõ ràng và nhất quán, giúp dễ hiểu và dễ quản lý.
8. **Sử dụng Hypermedia (HATEOAS):** Sử dụng Hypermedia as the Engine of Application State (HATEOAS) để cung cấp thông tin về các hành động khả dụng cho tài nguyên trong phản hồi API. Điều này giúp client khám phá và tương tác với API một cách linh hoạt.

### 3.2 Đánh giá theo từng nguyên tắc

Đánh giá bản cũ: **Đạt**, **Một phần** hoặc **Chưa đạt**.

| # | Nguyên tắc | Bản cũ | Hiện tại | Sửa lỗi |
| :--- | :--- | :--- | :--- | :--- |
| 1 | Danh từ trong URL | Một phần | Toàn bộ là danh từ. Đăng nhập, refresh, đăng xuất lần lượt là tạo, xoay vòng và thu hồi token. Khoá tài khoản là `PUT .../status`. | L1, L2, L4 |
| 2 | HTTP status code | Một phần | Tạo token luôn trả 201. Thêm 405 kèm `Allow`, 406 và 415. Tài nguyên của người khác vẫn trả 404. | L5, L6, L7 |
| 3 | Versioning | Đạt | Giữ `/api/v1`. Thay route ngay trong v1, vì chỉ có một client. | Không có |
| 4 | Nhiều định dạng | Chưa đạt | JSON và XML, chọn bằng `Content-Type` và `Accept`, kể cả body lỗi. Parser XML chặn `DOCTYPE` và `ENTITY`. | L7, L17 |
| 5 | Lọc và phân trang | Một phần | Mọi danh sách dùng `limit`, `cursor`, `sort` và `field=op:value`. Trường và toán tử nằm trong whitelist của từng route. Thêm `?ids=` để đọc và xoá theo lô, và thêm route phân trang tin nhắn. | L8, L9, L10, L11, L12, L13, L14 |
| 6 | Token authentication | Đạt ở BE | Giữ JWT và refresh token xoay vòng. FE giờ tự xoay token khi gặp 401. | L19 |
| 7 | Đặt tên thống nhất | Một phần | Danh từ số nhiều, kebab-case, field `snake_case`. Danh sách luôn trả `items`, `next_cursor`, `has_more`. | L3, L12 |
| 8 | HATEOAS | Chưa đạt | Thêm `_links` và điểm vào `GET /api/v1`. Link đổi theo trạng thái, ví dụ `lock` hoặc `unlock`. Có 26/34 route có `_links` (stream đặt trong event `done`). Trong 8 route còn lại, 4 route trả 204 nên không có body, và 4 route có body nhưng chưa có link: `/health/live`, `/admin/stats`, `/admin/health`, `DELETE /admin/users/:id/tokens`. | L18 |

### 3.3 Cải thiện ngoài 8 nguyên tắc

- **Chat không chạy lặp:** gửi lại cùng `Idempotency-Key` sẽ phát lại câu trả lời đã lưu và không chạy LLM. Nếu lượt trước đang chạy hoặc body khác thì trả 409. Nếu lượt trước lỗi thì được chạy lại. Sửa L15 ([D-050](docs/AI_chat_log/Vuong_chat_logs.md)).
- **`PUT` và `DELETE` gọi lặp không gây tác dụng phụ:** nếu trạng thái không đổi thì không ghi gì, không tăng `token_version` và không thêm audit. Sửa L16.
- **`PATCH` cho sửa một phần, `PUT` cho thay toàn bộ** ([D-048](docs/AI_chat_log/Vuong_chat_logs.md)).
- **Test contract phủ đủ route:** [`flow.e2e-spec.ts`](backend_rag/test/flow.e2e-spec.ts) giờ so đủ 34 route, kể cả admin, và kiểm tra URL không chứa động từ. Sửa L20.

---

## 4. Toàn bộ API hiện tại

Base URL là `/api/v1`, gồm **34 route**.

**Auth:** `—` là public, `JWT?` là tuỳ chọn (có token thì lưu lịch sử), `JWT` là bắt buộc. Mọi route trong §4.8 cần JWT và role `admin`.

**Rate limit:** tính theo IP. Route không ghi giới hạn dùng mức chung 120 request/phút.

### 4.1 Điểm vào (1)

| Route | Auth | Mục đích |
| :--- | :--- | :--- |
| `GET /` | — | Liệt kê link tới các tài nguyên gốc |

### 4.2 Tài khoản (4)

| Route | Auth | Mục đích | Rate limit |
| :--- | :--- | :--- | :--- |
| `POST /users` | — | Đăng ký, luôn tạo role `user` | 5/phút |
| `GET /users/me` | JWT | Xem hồ sơ của mình | |
| `PATCH /users/me` | JWT | Sửa `display_name` | |
| `PUT /users/me/password` | JWT | Đổi mật khẩu | 5/phút |

### 4.3 Token (3)

| Route | Auth | Mục đích | Rate limit |
| :--- | :--- | :--- | :--- |
| `POST /auth/tokens` | — | Đăng nhập, cấp cặp token | 10/phút |
| `POST /auth/token-rotations` | Refresh token trong body | Xoay vòng refresh token, dùng lại token cũ sẽ thu hồi cả family | 30/phút |
| `POST /auth/token-revocations` | JWT | Đăng xuất | |

### 4.4 RAG (2)

| Route | Auth | Mục đích | Rate limit |
| :--- | :--- | :--- | :--- |
| `POST /rag/chat` | JWT? | Hỏi đáp, trả toàn bộ câu trả lời. Hỗ trợ `Idempotency-Key`. | 20/phút |
| `POST /rag/chat/stream` | JWT? | Hỏi đáp qua SSE, `_links` nằm trong event `done`. Hỗ trợ `Idempotency-Key`. | 20/phút |

### 4.5 Phiên chat (6)

| Route | Auth | Mục đích | Tham số chính |
| :--- | :--- | :--- | :--- |
| `GET /chat-sessions` | JWT | Liệt kê hoặc đọc theo lô | `limit` (1–50, mặc định 10), `cursor`, `q`, `ids`, `sort` (`last_message_at`, `created_at`, `title`), filter `created_at`, `last_message_at`, `message_count` |
| `DELETE /chat-sessions` | JWT | Xoá mềm theo lô | `ids` (tối đa 50) |
| `GET /chat-sessions/:id` | JWT | Xem phiên, kèm tối đa 200 tin mới nhất | |
| `PATCH /chat-sessions/:id` | JWT | Đổi tiêu đề | Body `{ title }` |
| `DELETE /chat-sessions/:id` | JWT | Xoá mềm | |
| `GET /chat-sessions/:id/messages` | JWT | Liệt kê tin nhắn | `limit` (1–100, mặc định 50), `cursor`, `sort` (`created_at`) |

### 4.6 Paper (2)

| Route | Auth | Mục đích | Rate limit |
| :--- | :--- | :--- | :--- |
| `GET /papers` | — | Đọc theo lô, bắt buộc có `ids` (tối đa 20) | 30/phút |
| `GET /papers/:paper_id` | — | Xem một paper | |

### 4.7 Health (3)

| Route | Auth | Mục đích |
| :--- | :--- | :--- |
| `GET /health` | — | Trạng thái tổng quát, luôn trả 200 |
| `GET /health/live` | — | Kiểm tra tiến trình còn sống |
| `GET /health/ready` | — | Trả 503 khi Mongo hoặc Qdrant chết |

Ở production, `/health` và `/health/ready` ẩn chi tiết như model và GPU.

### 4.8 Admin (13)

Các lời gọi admin được ghi vào `admin_audit_logs`, kể cả route đọc. Riêng `PUT` không làm đổi gì thì không ghi.

| Route | Mục đích | Tham số chính |
| :--- | :--- | :--- |
| `GET /admin/users` | Liệt kê user | `limit`, `cursor`, `q`, `sort` (`created_at`, `email`), filter `role`, `is_active`, `email`, `created_at` |
| `GET /admin/users/:id` | Xem chi tiết user | |
| `PUT /admin/users/:id/roles` | Đặt lại roles | Body `{ roles }` |
| `PUT /admin/users/:id/status` | Khoá hoặc mở khoá | Body `{ active, reason? }` |
| `GET /admin/users/:id/tokens` | Xem các token family | |
| `DELETE /admin/users/:id/tokens` | Buộc đăng xuất | |
| `GET /admin/users/:id/chat-sessions` | Liệt kê phiên của user | `limit`, `cursor`, `q`, `sort`, filter `created_at` |
| `GET /admin/chat-sessions/:id` | Xem phiên, kèm tối đa 200 tin mới nhất | |
| `GET /admin/chat-sessions/:id/messages` | Liệt kê tin nhắn của phiên | `limit`, `cursor`, `sort` |
| `GET /admin/stats` | Xem thống kê | |
| `GET /admin/login-attempts` | Xem các lần đăng nhập | `limit`, `cursor`, `sort`, filter `created_at` |
| `GET /admin/health` | Xem health đầy đủ | |
| `GET /admin/audit-logs` | Xem nhật ký audit, không có route sửa hoặc xoá | `limit`, `cursor`, `sort`, filter `actor_id`, `action`, `target_type`, `created_at` |

Với admin, `limit` nhận 1–50 và mặc định là 20.

Ngoài 34 route trên, Swagger UI có ở `/api/v1/docs` và `/api/v1/docs-json`. Swagger bị tắt ở production.

---

## 5. Bảng đổi route

Các route cũ không còn tồn tại, và không có alias.

### 5.1 Đổi route (13 route cũ thành 12 route mới)

| Route cũ | Route hiện tại | Lý do |
| :--- | :--- | :--- |
| `POST /auth/register` | `POST /users` | Đăng ký là tạo user |
| `POST /auth/login` | `POST /auth/tokens` | Đăng nhập là tạo token |
| `POST /auth/refresh` | `POST /auth/token-rotations` | Bỏ động từ |
| `POST /auth/logout` | `POST /auth/token-revocations` | Refresh token phải đi trong body, mà `DELETE` không nên có body |
| `GET /auth/me` | `GET /users/me` | Tách tài khoản khỏi token |
| `PATCH /auth/me` | `PATCH /users/me` | Tách tài khoản khỏi token |
| `POST /auth/change-password` | `PUT /users/me/password` | Mật khẩu là tài nguyên con, thay toàn bộ |
| `PATCH /admin/users/:id/roles` | `PUT /admin/users/:id/roles` | Body là toàn bộ roles |
| `POST /admin/users/:id/disable` | `PUT /admin/users/:id/status` | Gộp 2 động từ thành một trạng thái |
| `POST /admin/users/:id/enable` | `PUT /admin/users/:id/status` | Gộp 2 động từ thành một trạng thái |
| `POST /admin/users/:id/revoke-tokens` | `DELETE /admin/users/:id/tokens` | Thu hồi là xoá tài nguyên token |
| `GET /admin/users/:id/sessions` | `GET /admin/users/:id/chat-sessions` | Cùng tên với phía user |
| `GET /admin/sessions/:sessionId` | `GET /admin/chat-sessions/:id` | Cùng tên với phía user |

### 5.2 Route mới (5)

| Route | Lý do |
| :--- | :--- |
| `GET /` | Điểm vào cho HATEOAS |
| `DELETE /chat-sessions` | Xoá theo lô |
| `GET /chat-sessions/:id/messages` | Đọc được tin cũ hơn 200 tin mới nhất |
| `GET /papers` | Đọc paper theo lô |
| `GET /admin/chat-sessions/:id/messages` | Đọc được tin cũ hơn 200 tin mới nhất ở phía admin |

### 5.3 Giữ path, đổi hành vi (17)

| Nhóm | Route | Thay đổi |
| :--- | :--- | :--- |
| Phiên chat | `GET /chat-sessions` | Thêm `limit`, `sort`, filter, `ids` |
| Phiên chat | `GET`, `PATCH`, `DELETE /chat-sessions/:id` | `GET` và `PATCH` thêm `_links`. Đổi tiêu đề giờ làm mới toàn bộ cache danh sách của user. |
| RAG | `POST /rag/chat`, `POST /rag/chat/stream` | Thêm `Idempotency-Key` và `_links` |
| Paper | `GET /papers/:paper_id` | Thêm `_links` |
| Health | `GET /health`, `GET /health/ready` | Thêm `_links` |
| Health | `GET /health/live` | Chỉ thêm XML và 405 như mọi route |
| Admin | `GET /admin/users` | Đổi sang query dùng chung, `active` thành `is_active`, sửa lỗi cursor |
| Admin | `GET /admin/login-attempts` | Thêm cursor, `sort`, filter và cấu trúc danh sách chung |
| Admin | `GET /admin/audit-logs` | Thêm `sort`, toán tử filter, `target_type`, `created_at` |
| Admin | `GET /admin/users/:id`, `GET /admin/users/:id/tokens` | Thêm `_links` |
| Admin | `GET /admin/stats`, `GET /admin/health` | Chỉ thêm XML và 405 như mọi route |

---

## 6. Hạn chế và việc còn mở

### 6.1 Chưa dùng được đầy đủ

- **Chat và paper chưa có nội dung thật:** payload Qdrant không còn `text`, nên [`vector-store.service.ts`](backend_rag/src/vector-store/vector-store.service.ts) bỏ qua mọi kết quả. Chat đang trả câu trả lời "không có ngữ cảnh" thay vì trích dẫn. BE cần đọc từ `rag_ts_corpus`, sau khi đã bù dữ liệu còn thiếu.
- **Chưa chạy k6 trên thiết kế mới:** kết quả 400 CU trước đây không còn giá trị, vì đổi tiêu đề giờ làm mới toàn bộ cache danh sách của user ([D-051](docs/AI_chat_log/Vuong_chat_logs.md)).

### 6.2 Điểm cố ý lệch khỏi nguyên tắc

| Điểm lệch | Lý do |
| :--- | :--- |
| Dùng `limit` và `cursor` thay vì `page` và `size` | Danh sách phiên chat đổi liên tục, nên offset dễ trùng hoặc sót dòng và chậm dần ở trang sâu ([D-049](docs/AI_chat_log/Vuong_chat_logs.md)). |
| `POST /rag/chat` không theo dạng danh từ số nhiều, và `/rag/chat/stream` đặt tên theo cách truyền | Một lượt chat không phải tài nguyên được đọc lại ở route này, và stream chỉ là cách trả khác của cùng thao tác. |
| `POST /auth/token-revocations` thay vì `DELETE` | Refresh token phải đi trong body. |
| `POST /auth/token-rotations` không idempotent | Refresh token chỉ dùng một lần, để phát hiện token bị đánh cắp. |
| `POST /rag/chat/stream` luôn trả `text/event-stream` | SSE không có dạng XML. Body request vẫn có thể là XML. |

### 6.3 Thiếu sót đã biết

- **HATEOAS mới dừng ở phía server:** FE vẫn hard-code đường dẫn thay vì đi theo `_links` ([D-053](docs/AI_chat_log/Vuong_chat_logs.md)), và schema Swagger chưa mô tả `_links`.
- **`Idempotency-Key` chỉ có tác dụng khi có JWT:** không có JWT thì không lưu gì, nên header bị bỏ qua.
- **Tài liệu lệch code ở 3 chỗ trong [`API_DESIGN.md`](docs/API_DESIGN.md):**
  - §2 ghi mật khẩu từ 8 ký tự, nhưng [`common-passwords.ts`](backend_rag/src/auth/common-passwords.ts) yêu cầu 10–200 ký tự.
  - §1.7 không liệt kê `DELETE /admin/users/:id/tokens` trong các route không có `_links`.
  - §5 và §7 còn mô tả dữ liệu paper lấy từ payload Qdrant.

### 6.4 Trạng thái kiểm thử

Các số dưới đây lấy từ lần chạy trước khi xử lý thay đổi dữ liệu. Lần này mình không chạy lại test.

| Phần | Kết quả |
| :--- | :--- |
| BE unit | 210/210 |
| BE e2e | 13/13 |
| FE Playwright | 6/6 |
| FE Vitest | 264/275, 11 lỗi có từ trước |
| FE và BE chạy thật | Đã qua: đăng ký, đăng nhập, đăng xuất, xoay token, khôi phục lịch sử, đổi tiêu đề, xoá phiên |
| Chưa kiểm chứng | Chat với LLM thật và trích dẫn, nội dung `/papers`, k6 |

FE còn một test chập chờn là `allows stopping an active stream`, chưa được sửa.
