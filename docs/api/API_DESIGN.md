# API_DESIGN.md — Thiết kế API (Monolith NestJS)

- **Motivation/Background**: API cũ lệch giữa FE, BE và docs (FE gửi `?limit=` bị trả 400, admin cursor hỏng ở trang 2, không có idempotency, tên route lẫn động từ). Tài liệu này là contract chuẩn duy nhất cho toàn bộ API.
- **Purpose**: Định nghĩa 8 tiêu chuẩn RESTful của dự án và mọi route của `/api/v1`; test `API contract (OpenAPI)` trong `backend_rag/test/flow.e2e-spec.ts` giữ code khớp với tài liệu này.
- **Overview Pipeline**: Danh từ cho URL → Status code → Versioning → Nhiều định dạng → Lọc & phân trang → Token auth → Đặt tên thống nhất → Hypermedia.
- **Detailed Plan**: §1 Quy ước (8 tiêu chuẩn, query language, định dạng, hypermedia); §2 Tài khoản & token; §3 RAG; §4 Phiên chat; §5 Paper; §6 Health; §7 Dữ liệu; §8 Ngoài phạm vi; §9 Admin.
- **References**: [API_SPECIFICATION.md](API_SPECIFICATION.md), [prompt_security.md](../prompts/prompt_security.md), [backend_rag/docs/CHAT_SESSIONS_SPECIFICATION.md](../../backend_rag/docs/CHAT_SESSIONS_SPECIFICATION.md).
- **Created**: 2026-10-20T18:01:49+07:00
- **Last Updated**: 2026-11-06T13:13:54+07:00

[STATUS: ACTIVE]

---

- **Kiến trúc**: 1 app NestJS. Mỗi tính năng là 1 module (`controller` + `service`).
- **Phạm vi**: RAG_TS chỉ **đọc** dữ liệu để hỏi đáp. Cào data, nạp dữ liệu và World 3D thuộc pipeline UTH (§8).
- **Stack**: MongoDB (user, token, session chat, idempotency key), Qdrant (vector + payload chunk, do pipeline ghi). Không dùng SQL.
- **Base URL**: `/api/v1`. Frontend đặt `VITE_API_BASE=/api/v1`.
- **Auth**: `Authorization: Bearer <access_token>`. Role: `user`, `admin`.
  - `—`: public · `JWT?`: tuỳ chọn (có token thì lưu lịch sử) · `JWT`: bắt buộc · `Admin`: JWT + role `admin`.

---

## 1. Quy ước chung

### 1.1 Status code

| Code | Khi nào | `message` mẫu |
| :--- | :--- | :--- |
| 200 | Đọc hoặc cập nhật thành công | — |
| 201 | Tạo mới (user, cặp token) | — |
| 204 | Thành công, không có body | — |
| 400 | Sai validate, sai `sort` / filter / `cursor` / `ids` | `invalid filter: role` |
| 401 | Thiếu / sai / hết hạn token | `unauthorized`, `token expired` |
| 403 | Không đủ quyền | `admin role required` |
| 404 | Không tìm thấy, hoặc tài nguyên của người khác | `session not found` |
| 405 | Path có tồn tại nhưng sai method (kèm header `Allow`) | `method not allowed` |
| 406 | `Accept` không cho phép JSON hoặc XML | `accept must allow application/json or application/xml` |
| 409 | Xung đột nghiệp vụ / trạng thái | `email already exists`, `request already in progress` |
| 415 | Body không phải JSON hoặc XML | `content type must be application/json or application/xml` |
| 429 | Quá giới hạn | `too many requests`, `llm queue is full` |
| 500 | Lỗi không lường trước | `internal server error` |
| 503 | Phụ thuộc chết (Mongo, Qdrant, LLM) | `qdrant unavailable` |
| 504 | LLM quá thời gian | `llm timeout` |

### 1.2 Body lỗi

```json
{ "statusCode": 404, "error": "Not Found", "message": "session not found", "request_id": "0199f0c2-..." }
```

`request_id` cũng nằm ở header `X-Request-Id`.

### 1.3 Log

Mỗi request ghi 1 dòng JSON khi kết thúc. Một số sự kiện nghiệp vụ ghi thêm dòng riêng.

- `info` cho 2xx, `warn` cho 4xx, `error` cho 5xx (kèm `stack`).
- Không log: password, token, nội dung câu hỏi (chỉ log `query_len`).

### 1.4 Tám tiêu chuẩn

| # | Tiêu chuẩn | Áp dụng trong dự án |
| :--- | :--- | :--- |
| 1 | Danh từ cho URL | `/users`, `/chat-sessions`, `/auth/tokens`. Đăng nhập, refresh, đăng xuất lần lượt là tạo, xoay vòng và thu hồi token. |
| 2 | HTTP status code | Theo §1.1. Tài nguyên của người khác luôn là 404, không phải 403. |
| 3 | Versioning | Trên path: `/api/v1`. Đổi contract mà phá tương thích thì lên `/api/v2`. |
| 4 | Nhiều định dạng | JSON và XML, chọn bằng `Content-Type` và `Accept` (§1.6). |
| 5 | Lọc và phân trang | Mọi danh sách dùng chung `limit`, `cursor`, `sort`, filter (§1.5). |
| 6 | Token authentication | JWT access token kèm refresh token xoay vòng, gửi qua `Authorization: Bearer`. |
| 7 | Đặt tên rõ ràng, thống nhất | Danh từ số nhiều, kebab-case; field `snake_case`; danh sách luôn trả `items`, `next_cursor`, `has_more`. |
| 8 | Hypermedia (HATEOAS) | `_links` trong response và điểm vào `GET /api/v1` (§1.7). |

**Quy tắc bổ sung của dự án:**

- **HTTP method**: `GET` đọc, `POST` tạo, `PATCH` sửa một phần (`/users/me`, `/chat-sessions/:id`), `PUT` thay toàn bộ (`roles`, `status`, `password`), `DELETE` xoá.
- **Idempotency**: `GET`, `PUT`, `PATCH`, `DELETE` gọi lặp không đổi kết quả. `PUT` và `DELETE` không đổi trạng thái thì không ghi gì thêm. `POST /rag/chat*` dùng `Idempotency-Key` (§3). `POST /auth/token-rotations` **cố ý không** idempotent: refresh token dùng một lần, dùng lại sẽ thu hồi cả family.
- **Batch**: đọc bằng `GET /chat-sessions?ids=a,b` và `GET /papers?ids=a,b`; xoá bằng `DELETE /chat-sessions?ids=a,b`.

### 1.5 Query language

```
GET /chat-sessions?limit=20&sort=title:asc,created_at:desc&created_at=gte:2026-01-01,lt:2026-02-01
```

- **Phân trang**: `limit` + `cursor` → `{ "items": [...], "next_cursor": "..." | null, "has_more": false }`. Dùng cursor thay `page`/`size` vì danh sách phiên chat thay đổi liên tục: offset sẽ trùng hoặc sót dòng và chậm dần ở trang sâu.
- **Sort**: `field:asc|desc`, tối đa 3 trường, chỉ trường được liệt kê ở từng route. `_id` luôn là tie-break. Cursor gắn với sort đã tạo ra nó: đổi `sort` mà giữ cursor → 400 `invalid cursor`.
- **Filter**: `field=op:value`, nhiều phép so sánh nối bằng dấu phẩy. Giá trị trần tương đương `eq:`. Toán tử: `eq`, `ne`, `gt`, `gte`, `lt`, `lte` và `match` (chuỗi con, không phân biệt hoa thường, theo nghĩa đen). Chỉ trường và toán tử trong whitelist của route mới tới được MongoDB.
- **`q`**: tìm tự do trên các trường text của route đó.

### 1.6 Định dạng dữ liệu

| Hướng | Header | Giá trị |
| :--- | :--- | :--- |
| Request | `Content-Type` | `application/json`, `application/xml`, `text/xml`. Khác → 415. |
| Response | `Accept` | Mặc định JSON. `application/xml` hoặc `text/xml` → XML, cả với body lỗi. Không chấp nhận được loại nào → 406. |

- XML: một phần tử gốc tuỳ tên ở request, gốc `<response>` ở response. Danh sách lặp lại tag (`<items>...</items><items>...</items>`); danh sách rỗng không có tag.
- Chặn `DOCTYPE` và `ENTITY` (XXE, billion laughs); XML sai → 400 `invalid xml`. Body tối đa 100 KB.
- `POST /rag/chat/stream` luôn là `text/event-stream`, body request vẫn có thể là XML. `GET /health/live` trả theo `Accept` như mọi route khác.

### 1.7 Hypermedia

Dạng link: `{ "href": "/api/v1/...", "method": "PATCH" }`; không có `method` nghĩa là `GET`. `GET /api/v1` liệt kê các tài nguyên gốc, client chỉ cần biết URL này.

```jsonc
{
  "items": [
    { "id": "...", "title": "...", "_links": {
        "self": { "href": "/api/v1/chat-sessions/019..." },
        "messages": { "href": "/api/v1/chat-sessions/019.../messages" },
        "rename": { "href": "/api/v1/chat-sessions/019...", "method": "PATCH" },
        "delete": { "href": "/api/v1/chat-sessions/019...", "method": "DELETE" },
        "chat": { "href": "/api/v1/rag/chat", "method": "POST" } } }
  ],
  "next_cursor": "...", "has_more": true,
  "_links": { "self": { "href": "..." }, "next": { "href": "...&cursor=..." } }
}
```

- Danh sách có `self` và `next` (khi còn trang). Link hành động theo trạng thái: admin user có `lock` khi đang hoạt động, `unlock` khi đang khoá.
- Câu trả lời chat có `session`, `messages`, `papers`; ở stream, các link nằm trong event `done`.
- Không có `_links`: `GET /health/live`, `GET /admin/stats`, `GET /admin/health`. Schema Swagger hiện chưa mô tả `_links`.

---

## 2. Tài khoản & token

Tài khoản ở `UsersController` (`/users`), vòng đời token ở `AuthController` (`/auth`).

| Route | Auth | Request | Response | Status · message | Log |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `POST /users` (đăng ký, 5/phút) | — | `RegisterReq` | `AuthRes` | 201 · 400 `invalid email` / `password too short` · 409 `email already exists` | `REGISTER_OK` |
| `GET /users/me` | JWT | — | `User` | 200 · 401 | — |
| `PATCH /users/me` | JWT | `{ display_name }` | `User` | 200 · 400 `invalid display_name` | `PROFILE_UPDATED` |
| `PUT /users/me/password` (5/phút) | JWT | `{ old_password, new_password }` | — | 204 · 401 `old password incorrect` · 400 `password too short` | `PASSWORD_CHANGED` |
| `POST /auth/tokens` (đăng nhập, 10/phút) | — | `{ email, password }` | `AuthRes` | 201 · 401 `invalid credentials` · 429 `too many login attempts` | `LOGIN_OK` / `LOGIN_FAILED` |
| `POST /auth/token-rotations` (refresh, 30/phút) | — | `{ refresh_token }` | `AuthRes` | 201 · 401 `refresh token invalid or expired` | `REFRESH_OK` / `REFRESH_REUSED` |
| `POST /auth/token-revocations` (đăng xuất) | JWT | `{ refresh_token }` | — | 204 · 401 | `LOGOUT_OK` |

Đăng xuất là `POST` vì refresh token phải đi trong body, và `DELETE` không nên có body.

```jsonc
// RegisterReq — password ≥ 8 ký tự
{ "email": "a@example.com", "password": "********", "display_name": "Manh" }

// AuthRes — access 15 phút, refresh 7 ngày, xoay vòng mỗi lần refresh
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 900, "refresh_token": "...", "user": { /* User */ } }

// User
{ "id": "...", "email": "a@example.com", "display_name": "Manh", "roles": ["user"], "created_at": "..." }
```

---

---

## 3. RAG — `RagController` (`/rag`)

Giới hạn 20 request/phút mỗi IP, vì LLM queue chỉ có một slot.

**Idempotency** (chỉ khi có JWT; không có JWT thì không lưu gì nên header bị bỏ qua). Client gửi `Idempotency-Key: <uuid>`; key sống 24 giờ:

| Tình huống gửi lại cùng key | Kết quả |
| :--- | :--- |
| Lần trước đã xong | Phát lại câu trả lời đã lưu, không chạy LLM, không lưu thêm; header `Idempotent-Replayed: true`. Với stream: `retrieval` → `phase` → `token` → `done`. |
| Lần trước đang chạy | 409 `request already in progress` |
| Body khác | 409 `idempotency key reused with a different request` |
| Lần trước lỗi / bị huỷ | Chạy lại bình thường (key đã được trả) |
| Worker chết giữa chừng | Chạy lại được sau khi khoá 5 phút hết hạn |
| Key không phải UUID | 400 `invalid Idempotency-Key` |


| Route | Auth | Request | Response | Status · message | Log |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `POST /rag/chat` | JWT? | `ChatReq` | `ChatRes` | 200 · 400 `validation error` · 404 `session not found` · 429 `llm queue is full` · 503 `qdrant unavailable` / `llm unavailable` · 504 `llm timeout` | `CHAT_OK` (kèm `query_len`, `top_k`, `chunks`, `timings`) / `CHAT_FAILED` |
| `POST /rag/chat/stream` | JWT? | `ChatReq` | SSE (§3.1) | 200 + stream · lỗi trước khi stream: như trên · lỗi giữa stream: event `error` | `CHAT_STREAM_OK` / `CHAT_STREAM_ABORTED` (client đóng) / `CHAT_FAILED` |

```jsonc
// ChatReq
{
  "query": "How does LoRA optimize large models?", // bắt buộc, 1–2000 ký tự
  "top_k": 5,                  // 1–20, mặc định 5
  "nprobe": 2,                 // 1–6, mặc định 2: số cành / nhánh giữ lại mỗi tầng cây (event route)
  "category": "cs.LG",         // tuỳ chọn, lọc theo payload Qdrant
  "use_authority_boost": true, // mặc định true: final = 0.85·sim + 0.15·authority
  "expand_query": false,       // mặc định false: mở rộng bằng luật FP-Growth
  "session_id": "..."          // tuỳ chọn, cần JWT; không có thì tạo phiên mới
}

// ChatRes
{
  "session_id": "...",        // null nếu không có JWT
  "message_id": "...",
  "query": "...",
  "answer": "LoRA freezes ... [Paper: 2106.09685]",
  "citations": ["2106.09685"],
  "grounded_chunks": [ /* GroundedChunk */ ],
  "expanded_terms": [],
  "timings": { "embed_ms": 35, "route_ms": 2, "search_ms": 18, "rerank_ms": 1, "llm_ms": 1600, "total_ms": 1660 }
}

// GroundedChunk
{ "chunk_id": "2106.09685_sec8", "paper_id": "2106.09685", "title": "...", "section_title": "Method", "section_type": "method", "text": "...", "similarity_score": 0.53, "authority_score": 0.31, "final_score": 0.49, "citation_tag": "[Paper: 2106.09685]" }
```

### 3.1 Sự kiện SSE của `/rag/chat/stream`

Header: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `X-Accel-Buffering: no`. Mỗi event là `data: <json>`, phân biệt bằng `type`. Cứ 15 giây gửi `: ping`.

| Thứ tự | `type` | `data` |
| :--- | :--- | :--- |
| 1 | `phase` | `{ "phase": "embedding" \| "flying" \| "comparing" \| "found" \| "generating" }` (gửi mỗi lần đổi giai đoạn) |
| 2 | `route` | `{ "steps": [{ "depth": 1, "candidates": [{ "node_id": 3, "score": 0.71 }], "chosen": [3] }] }` |
| 3 | `compare` | `{ "candidates": [{ "paper_id": "...", "score": 0.62 }] }` |
| 4 | `retrieval` | `{ "grounded_chunks": [...], "expanded_terms": [] }` |
| 5 | `token` | `{ "text": "LoRA" }` (0..n lần) |
| 6 | `done` | `{ "citations": [...], "timings": {...}, "session_id": "...", "message_id": "..." }` |
| — | `error` | `{ "message": "llm timeout", "status": 504 }` (thay cho `done`) |

Client đóng kết nối thì server huỷ lượt sinh LLM và không lưu message.

---

## 4. Phiên chat — `ChatSessionsController` (`/chat-sessions`)

Tất cả cần JWT. Phiên của người khác trả **404** (không lộ là có tồn tại). Phiên được tạo khi gửi tin nhắn đầu tiên qua `/rag/chat*`, nên không có `POST /chat-sessions`.

| Route | Request | Response | Status · message | Log |
| :--- | :--- | :--- | :--- | :--- |
| `GET /chat-sessions` | `limit` (1–50, mặc định 10), `cursor`, `q`, `ids` (tối đa 50), `sort` (`last_message_at`, `created_at`, `title`; mặc định `last_message_at:desc`), filter `created_at`, `last_message_at`, `message_count` | `Page<SessionSummary>` | 200 · 400 | — |
| `DELETE /chat-sessions?ids=a,b` | tối đa 50 id | — | 204 (gọi lặp vẫn 204; id lạ bị bỏ qua) · 400 `invalid ids` | `SESSIONS_DELETED` |
| `GET /chat-sessions/:id` | — | `Session` (kèm tối đa 200 tin nhắn mới nhất) | 200 · 404 `session not found` | — |
| `PATCH /chat-sessions/:id` | `{ title }` (trim, 1–120 ký tự) | `SessionSummary` | 200 · 400 `invalid title` · 404 | `SESSION_RENAMED` |
| `DELETE /chat-sessions/:id` | — | — | 204 (gọi lặp vẫn 204) | `SESSION_DELETED` |
| `GET /chat-sessions/:id/messages` | `limit` (1–100, mặc định 50), `cursor`, `sort` (`created_at`; mặc định `created_at:asc`) | `Page<Message>` | 200 · 404 | — |

```jsonc
// SessionSummary — title mặc định là 60 ký tự đầu của câu hỏi đầu tiên
{ "id": "...", "title": "LoRA và fine-tuning", "message_count": 4, "created_at": "...", "updated_at": "..." }

// Session
{
  "id": "...", "title": "...", "created_at": "...", "updated_at": "...",
  "messages": [
    { "id": "...", "role": "user", "content": "How does LoRA ...?", "created_at": "..." },
    { "id": "...", "role": "assistant", "content": "...", "citations": ["2106.09685"], "grounded_chunks": [...], "timings": {...}, "created_at": "..." }
  ]
}
```

---

## 5. Paper — `PapersController` (`/papers`)

`GET /papers?ids=a,b,c` đọc theo lô, tối đa 20 id, 30 request/phút → `{ "items": PaperDetail[], "not_found": string[] }`. Không có danh sách toàn bộ: thiếu `ids` trả 400. Qdrant chết thì cả request trả 503.

| Route | Auth | Request | Response | Status · message | Log |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `GET /papers/:paper_id` | — | — | `PaperDetail` | 200 · 404 `paper not found` · 503 `qdrant unavailable` | — |

`paper_id` có thể chứa `.` và `:` (`2310.00297`, `openalex:W2594538354`), client phải URL-encode.

```jsonc
// PaperDetail
{ "paper_id": "...", "title": "...", "authors": [...], "category_index": 2, "primary_category": "cs.CL", "year": 2021, "abstract": "...", "has_fulltext": true,
  "chunks": [{ "chunk_id": "...", "section_title": "Introduction", "section_type": "introduction", "text": "..." }] }
```

Không lưu paper ở MongoDB. `PaperDetail` được dựng từ Qdrant: `scroll` với filter `paper_id`, gom các chunk lại. `title`, `authors`, `primary_category`, `year` lấy từ chunk đầu tiên. `abstract` là `text` của chunk có `section_type = "abstract"`. `has_fulltext` là true khi có chunk khác abstract. `category_index` tra từ `primary_category`.

---

## 6. Health — `HealthController` (`/health`)

| Route | Response | Status |
| :--- | :--- | :--- |
| `GET /health` | `Health` | Luôn 200, xem `status` |
| `GET /health/live` | `{ "status": "ok" }` | 200 nếu tiến trình sống |
| `GET /health/ready` | `Health` | 200 khi `ok` / `degraded` · 503 khi `down` (Mongo hoặc Qdrant chết) |

```jsonc
// Health — LLM chết thì "degraded" (vẫn xem paper được)
{ "status": "ok", "version": "1.0.0", "uptime_s": 3600,
  "world_version": "w-2026-10-20", // null khi chưa nạp file world
  "components": { "mongo": true, "qdrant": true, "llm": true, "world": true }, // world: đã nạp cây + centroid, không ảnh hưởng status
  "llm": { "model_id": "qwen2.5-7b-instruct-q4_k_m", "gpu": "cuda", "queue_length": 0 } }
```

Log: khi một component đổi trạng thái thì ghi `HEALTH_CHANGED` (`warn`), kèm `component`, `from`, `to`.

---

## 7. Dữ liệu

| Store | Collection | Field chính | Index |
| :--- | :--- | :--- | :--- |
| MongoDB | `users` | `email`, `password_hash` (argon2), `display_name`, `roles` | `email` unique |
| MongoDB | `refresh_tokens` | `user_id`, `token_hash`, `family_id`, `revoked_at`, `expires_at` | `token_hash` unique, TTL `expires_at` |
| MongoDB | `chat_sessions` | `user_id`, `title`, `message_count`, `created_at`, `updated_at` | `user_id + updated_at` |
| MongoDB | `chat_messages` | `session_id`, `role`, `content`, `citations`, `grounded_chunks`, `timings` | `session_id + created_at` |
| Qdrant | `scientific_papers_gold` (pipeline ghi, RAG_TS chỉ đọc) | vector 768-D (nomic v1.5, cosine); payload `chunk_id`, `paper_id`, `title`, `authors`, `primary_category`, `year`, `section_title`, `section_type`, `text` | payload index `paper_id`, `primary_category`, `year` (integer) |

---

## 8. Ngoài phạm vi — do pipeline UTH cung cấp

Repo `UTH-Data-Mining-.../backend` (FastAPI, prefix `/api`) lo cào, embed, ghi Qdrant và dữ liệu mining. Frontend gọi thẳng pipeline cho 2 tính năng sau, RAG_TS không làm:

| Tính năng FE | Endpoint pipeline | Ghi chú |
| :--- | :--- | :--- |
| Hiệu ứng cào (HUD crawl) | `POST /api/ingestion/start`, `POST /api/ingestion/stop`, `GET /api/ingestion/status` | Start/stop chạy DAG harvest trên Airflow |
| Luồng cào realtime | `GET /api/ingestion/stream` (SSE) | Event `CONNECTION_ESTABLISHED`, `SNAPSHOT`, `LOG`, `PAPER_INGESTED`. FE đang chờ `crawl_status` / `crawl_batch` / `crawl_vectors` → cần lớp map ở `crawlClient.ts` |
| World 3D | `GET /api/mining/pillars/clusters` (K-Means/DBSCAN + toạ độ **2D**), `GET /api/mining/pillars/graph` | FE cần toạ độ 3D và `ground`, `tree`, `vectors.bin` → pipeline chưa có, cần bổ sung |

---

## 9. Admin — `AdminController` (`/admin`)

Mọi route cần JWT + role `admin`, và service kiểm tra lại role lần nữa. Mỗi lời gọi được ghi vào `admin_audit_logs` (append-only), trừ các `PUT` không làm đổi gì.

| Route | Request | Response | Ghi chú |
| :--- | :--- | :--- | :--- |
| `GET /admin/users` | `limit`, `cursor`, `q`, `sort` (`created_at`, `email`), filter `role`, `is_active`, `email`, `created_at` | `Page<AdminUser>` | |
| `GET /admin/users/:id` | — | `AdminUserDetail` | 404 `user not found` |
| `PUT /admin/users/:id/roles` | `{ roles }` | `AdminUser` | 409 `SELF_LOCKOUT_FORBIDDEN` / `LAST_ADMIN_PROTECTED`. Roles không đổi thì không ghi gì và không thu hồi token. |
| `PUT /admin/users/:id/status` | `{ active, reason? }` | `AdminUser` | Thay `disable` / `enable`. Trạng thái không đổi thì không ghi gì. Khoá lại với `reason` khác chỉ cập nhật `reason`. `reason` khi mở khoá → 400. |
| `GET /admin/users/:id/tokens` | — | `{ items: TokenFamily[] }` | |
| `DELETE /admin/users/:id/tokens` | — | `{ revoked_count }` | Force logout, luôn tăng `token_version`. |
| `GET /admin/users/:id/chat-sessions` | `limit`, `cursor`, `q`, `sort`, filter `created_at` | `Page<AdminSession>` | |
| `GET /admin/chat-sessions/:id` | — | `AdminSessionDetail` | Tối đa 200 tin nhắn mới nhất. |
| `GET /admin/chat-sessions/:id/messages` | `limit`, `cursor`, `sort` | `Page<AdminMessage>` | |
| `GET /admin/stats` | — | `AdminStats` | |
| `GET /admin/login-attempts` | `limit`, `cursor`, `sort`, filter `created_at` | `Page<LoginAttempt>` | |
| `GET /admin/health` | — | `Health` | Chi tiết đầy đủ, không che như `/health`. |
| `GET /admin/audit-logs` | `limit`, `cursor`, `sort`, filter `actor_id`, `action`, `target_type`, `created_at` | `Page<AuditLog>` | Không có route sửa hoặc xoá. |
