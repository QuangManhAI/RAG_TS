# API_DESIGN.md — Thiết kế API (Monolith NestJS)

- **Kiến trúc**: 1 app NestJS. Mỗi tính năng là 1 module (`controller` + `service`).
- **Phạm vi**: RAG_TS chỉ **đọc** dữ liệu để hỏi đáp. Cào data, nạp dữ liệu và World 3D thuộc pipeline UTH (§8).
- **Stack**: MongoDB (user, token, session chat), Qdrant (vector + payload chunk, do pipeline ghi). Không dùng SQL.
- **Base URL**: `/api/v1`. Frontend đặt `VITE_API_BASE=/api/v1`.
- **Auth**: `Authorization: Bearer <access_token>`. Role: `user`, `admin`.
  - `—`: public · `JWT?`: tuỳ chọn (có token thì lưu lịch sử) · `JWT`: bắt buộc · `Admin`: JWT + role `admin`.

## Mục lục

| # | Tính năng | Controller | Số API |
| :--- | :--- | :--- | :--- |
| 2 | Xác thực, tài khoản | `AuthController` | 7 |
| 3 | Hỏi đáp RAG | `RagController` | 2 |
| 4 | Phiên chat | `ChatSessionController` | 5 |
| 5 | Paper | `PaperController` | 1 |
| 6 | Health | `HealthController` | 3 |
| 7 | Dữ liệu | — | — |
| 8 | Ngoài phạm vi (pipeline UTH) | — | — |

---

## 1. Quy ước chung

### 1.1 Status code

| Code | Khi nào | `message` mẫu |
| :--- | :--- | :--- |
| 200 | Thành công | — |
| 201 | Tạo mới | — |
| 204 | Thành công, không có body | — |
| 400 | Sai validate | `query should not be empty` |
| 401 | Thiếu / sai / hết hạn token | `unauthorized`, `token expired` |
| 403 | Không đủ quyền | `admin role required` |
| 404 | Không tìm thấy | `session not found` |
| 409 | Trùng / xung đột trạng thái | `email already exists` |
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

Mỗi request ghi 1 dòng JSON khi kết thúc. Một số sự kiện nghiệp vụ ghi thêm dòng riêng (cột **Log** trong các bảng dưới).

```json
{ "level": "info", "time": "2026-10-20T08:00:00.000Z", "request_id": "...", "method": "POST", "path": "/api/v1/auth/login", "status": 200, "latency_ms": 42, "user_id": "...", "msg": "LOGIN_OK" }
```

- `info` cho 2xx, `warn` cho 4xx, `error` cho 5xx (kèm `stack`).
- Không log: password, token, nội dung câu hỏi (chỉ log `query_len`).

### 1.4 Phân trang

Danh sách dùng cursor: `?limit=20&cursor=<opaque>` (`limit` 1–50) → `{ "items": [...], "next_cursor": "..." | null }`.

---

## 2. Auth — `AuthController` (`/auth`)

| Route | Auth | Request | Response | Status · message | Log |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `POST /auth/register` | — | `RegisterReq` | `AuthRes` | 201 · 400 `invalid email` / `password too short` · 409 `email already exists` | `REGISTER_OK` / `REGISTER_EMAIL_TAKEN` |
| `POST /auth/login` | — | `{ email, password }` | `AuthRes` | 200 · 401 `invalid credentials` · 429 `too many login attempts` | `LOGIN_OK` / `LOGIN_FAILED` (kèm `email_hash`) |
| `POST /auth/refresh` | — | `{ refresh_token }` | `AuthRes` | 200 · 401 `refresh token invalid or expired` | `REFRESH_OK` / `REFRESH_REUSED` (token đã thu hồi bị dùng lại → thu hồi cả họ token) |
| `POST /auth/logout` | JWT | `{ refresh_token }` | — | 204 · 401 | `LOGOUT_OK` |
| `GET /auth/me` | JWT | — | `User` | 200 · 401 | — |
| `PATCH /auth/me` | JWT | `{ display_name }` (1–60 ký tự) | `User` | 200 · 400 `invalid display_name` | `PROFILE_UPDATED` |
| `POST /auth/change-password` | JWT | `{ old_password, new_password }` | — | 204 · 400 `password too short` · 401 `old password incorrect` | `PASSWORD_CHANGED` (thu hồi mọi refresh token) |

```jsonc
// RegisterReq — password ≥ 8 ký tự
{ "email": "a@example.com", "password": "********", "display_name": "Manh" }

// AuthRes — access 15 phút, refresh 7 ngày, xoay vòng mỗi lần refresh
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 900, "refresh_token": "...", "user": { /* User */ } }

// User
{ "id": "...", "email": "a@example.com", "display_name": "Manh", "roles": ["user"], "created_at": "..." }
```

---

## 3. RAG — `RagController` (`/rag`)

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

## 4. Phiên chat — `ChatSessionController` (`/chat-sessions`)

Tất cả cần JWT. Phiên của người khác trả **404** (không lộ là có tồn tại).

| Route | Request | Response | Status · message | Log |
| :--- | :--- | :--- | :--- | :--- |
| `GET /chat-sessions?limit&cursor&q` | `q`: tìm theo tiêu đề | `{ items: SessionSummary[], next_cursor }` | 200 · 400 `invalid cursor` | — |
| `POST /chat-sessions` | `{ title? }` | `SessionSummary` | 201 · 400 `invalid title` | `SESSION_CREATED` |
| `GET /chat-sessions/:id` | — | `Session` | 200 · 404 `session not found` | — |
| `PATCH /chat-sessions/:id` | `{ title }` (trim, 1–120 ký tự) | `SessionSummary` | 200 · 400 `invalid title` · 404 | `SESSION_RENAMED` |
| `DELETE /chat-sessions/:id` | — | — | 204 (gọi lặp vẫn 204) | `SESSION_DELETED` |

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

## 5. Paper — `PaperController` (`/papers`)

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
