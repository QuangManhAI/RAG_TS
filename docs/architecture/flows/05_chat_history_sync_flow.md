# Luồng 05 — Đồng bộ Lịch sử Hội thoại Offline-First & Phân trang UUIDv7 Cursor (GET /api/v1/chat-sessions)

- **Motivation/Background**: Trong nền tảng `RAG_TS`, người dùng cần xem lại toàn bộ các phiên trò chuyện khoa học (`chat_sessions`) và lịch sử tin nhắn (`chat_messages`) tức thì ngay cả khi máy chủ ngoại tuyến hoặc đang chạy qua đường hầm mạng (`Cloudflare Access TCP Tunnel` tới MongoDB từ xa tốn ~15–25ms mỗi round-trip). Đồng thời, cơ chế phân trang phải tránh lỗi bỏ sót/trùng lặp dữ liệu khi có tin nhắn mới xen vào và bảo vệ tuyệt đối quyền riêng tư giữa các tài khoản bằng khóa chính `UUIDv7` và kiểm tra quyền sở hữu tầng ứng dụng.
- **Purpose**: Tài liệu hóa chi tiết luồng request-response đầu-cuối (end-to-end) cho kiến trúc đồng bộ lịch sử hội thoại Offline-First (`Cache-First` với `Dexie.js` IndexedDB) kết hợp phân trang `UUIDv7` Keyset Cursor qua hai endpoint chính `GET /api/v1/chat-sessions` và `GET /api/v1/chat-sessions/:id/messages` (cùng nhánh `Singleflight` của `GET /api/v1/chat-sessions/:id`).
- **Overview Pipeline**: Mount/Auth (`ChatPanel`) → `useSessionStore.init()` đọc `Dexie.js` tức thì → `GET /api/v1/health` → `fetchAllPages` phân trang `GET /api/v1/chat-sessions` → Backend `LRUCache` theo `userGen` & Keyset Query MongoDB `.lean()` → Hợp nhất Last-Write-Wins vào `RagChatDB` → Chọn phiên (`switchSession`) tải tin nhắn từ IndexedDB hoặc lazy-fetch `GET /api/v1/chat-sessions/:id/messages` qua `Promise.all` → Hydrate `useRag.loadMessages()`.
- **Detailed Plan**: Mục 1 tổng quan kiến trúc Offline-First & phân trang Keyset; Mục 2 sơ đồ luồng thành phần (`flowchart TB`) và tuần tự (`sequenceDiagram`); Mục 3 phân rã chi tiết 4 giai đoạn kèm HTTP Wire Examples; Mục 4 đường dẫn lỗi và các bất biến hệ thống (`System Invariants`).
- **References**: [ChatPanel.tsx](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx), [HistoryDrawer.tsx](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx), [useChatWorkspace.ts](../../../frontend_rag/src/features/chat/hooks/useChatWorkspace.ts), [sessionStore.ts](../../../frontend_rag/src/features/chat/store/sessionStore.ts), [ragStore.ts](../../../frontend_rag/src/features/chat/store/ragStore.ts), [chatSessionService.ts](../../../frontend_rag/src/features/chat/services/chatSessionService.ts), [chatApi.ts](../../../frontend_rag/src/features/chat/api/chatApi.ts), [chatDb.ts](../../../frontend_rag/src/features/chat/db/chatDb.ts), [messageMapping.ts](../../../frontend_rag/src/features/chat/lib/messageMapping.ts), [http.ts](../../../frontend_rag/src/shared/api/http.ts), [config.ts](../../../frontend_rag/src/shared/api/config.ts), [app.setup.ts](../../../backend_rag/src/app.setup.ts), [jwt-auth.guard.ts](../../../backend_rag/src/modules/auth/guards/jwt-auth.guard.ts), [pagination.ts](../../../backend_rag/src/common/pagination/pagination.ts), [sanitize.ts](../../../backend_rag/src/common/security/sanitize.ts), [hateoas.ts](../../../backend_rag/src/common/http/hateoas.ts), [validation.ts](../../../backend_rag/src/common/http/validation.ts), [xml.ts](../../../backend_rag/src/common/http/xml.ts), [chat-sessions.controller.ts](../../../backend_rag/src/modules/chat-sessions/chat-sessions.controller.ts), [chat-sessions.service.ts](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts), [chat-session.mapper.ts](../../../backend_rag/src/modules/chat-sessions/chat-session.mapper.ts), [chat-session.dto.ts](../../../backend_rag/src/modules/chat-sessions/dto/chat-session.dto.ts), [chat-sessions.constants.ts](../../../backend_rag/src/modules/chat-sessions/chat-sessions.constants.ts), [chat-sessions.repository.ts](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts), [chat-messages.repository.ts](../../../backend_rag/src/modules/chat-sessions/repositories/chat-messages.repository.ts), [chat-session.schema.ts](../../../backend_rag/src/modules/chat-sessions/schemas/chat-session.schema.ts), [chat-message.schema.ts](../../../backend_rag/src/modules/chat-sessions/schemas/chat-message.schema.ts).
- **Created**: 2026-11-26T10:12:00+07:00
- **Last Updated**: 2026-11-26T10:12:00+07:00

[STATUS: ACTIVE]

---

## 1. Tổng quan Kiến trúc Đồng bộ & Phân trang

Luồng đồng bộ lịch sử hội thoại của `RAG_TS` được thiết kế xoay quanh hai nguyên lý cốt lõi:

1. **Offline-First & Cache-First với IndexedDB (`Dexie.js`)**:
   - Toàn bộ danh sách phiên (`sessions`) và tin nhắn (`messages`) được lưu cục bộ trong IndexedDB `RagChatDB` ([chatDb.ts](../../../frontend_rag/src/features/chat/db/chatDb.ts#L30-L43)).
   - Khi [`ChatPanel`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L14-L29) mount hoặc trạng thái đăng nhập thay đổi, `useSessionStore.init()` đọc ngay danh sách phiên từ `Dexie.js` để hiển thị tức thì (0ms network wait), sau đó mới kiểm tra `/health` và kéo dữ liệu mới từ Backend về hợp nhất theo `updated_at` (Last-Write-Wins).
   - Khi người dùng chuyển đổi phiên trong [`HistoryDrawer`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L24-L100), `loadSessionMessages()` ưu tiên đọc `chatDb.messages`; chỉ khi bảng cục bộ rỗng (ví dụ đăng nhập trên trình duyệt/thiết bị mới) và máy chủ đang online, Frontend mới gọi `GET /api/v1/chat-sessions/:id/messages` để lazy-load và cache xuống `Dexie.js`.
2. **Phân trang Keyset Cursor theo `UUIDv7` & Tối ưu độ trễ MongoDB**:
   - Cả `chat_sessions._id` và `chat_messages._id` đều dùng chuỗi `UUIDv7` sắp xếp theo thời gian kết hợp chỉ mục phức hợp `{ user_id: 1, deleted_at: 1, last_message_at: -1, _id: -1 }` và `{ session_id: 1, deleted_at: 1, created_at: -1, _id: -1 }`.
   - Backend truy vấn `limit + 1` bản ghi ở chế độ `.lean()` để xác định `has_more` và mã hóa con trỏ `next_cursor` dạng `base64url` gắn chặt với thứ tự `sort` (`{ s, v, id }`), tránh `skip()` chậm chạp và lỗi trượt trang.
   - Danh sách phiên được bảo vệ bởi `LRUCache` (`listCache`, tối đa `10_000` entry, TTL `300_000ms`) gắn khóa theo bộ đếm thế hệ từng người dùng (`userGen`), còn `listMessages` chạy song song `Promise.all([findOwned, findPageLean])` để chỉ mất đúng 1 RTT qua Cloudflare TCP Tunnel.

---

## 2. Sơ đồ Kiến trúc & Tuần tự End-to-End

### 2.1. Sơ đồ Luồng Thành phần (Component & Layer Flow)

```mermaid
flowchart TB
  subgraph ClientFE["Frontend Layer (React 19 + Zustand + Dexie.js)"]
    UI["ChatPanel / HistoryDrawer"]
    Hooks["useChatPanelState / useHistoryDrawerState"]
    Store["useSessionStore (init, switchSession) & useRag (loadMessages)"]
    Svc["chatSessionService (Cache-First, syncRemoteSessions, loadSessionMessages)"]
    Dexie[("IndexedDB: RagChatDB (sessions, messages)")]
    Api["chatApi (fetchAllRemoteSessions, fetchAllRemoteMessages)"]
    Http["http.ts (fetchAllPages, apiFetch, publicFetch)"]
  end

  subgraph Backend["Backend Layer (NestJS 11)"]
    MW["configureApp Middlewares (requestId, securityHeaders, metrics, xmlBody)"]
    Guard["JwtAuthGuard (bearerToken, verifyAccess, getActiveUser)"]
    Pipe["ValidationPipe (whitelist, forbidNonWhitelisted, transform)"]
    Ctrl["ChatSessionsController (list, listMessages, get)"]
    Service["ChatSessionsService (listCache + userGen, Promise.all, Singleflight)"]
    Repo["ChatSessionsRepository & ChatMessagesRepository (.lean)"]
    Interceptors["LinksInterceptor (HATEOAS _links) & ContentNegotiationInterceptor"]
  end

  DB[("MongoDB (chat_sessions, chat_messages)")]

  UI --> Hooks --> Store --> Svc
  Svc <--> Dexie
  Svc --> Api --> Http
  Http -->|"GET /api/v1/chat-sessions & :id/messages"| MW
  MW --> Guard --> Pipe --> Ctrl --> Service
  Service -->|"Cache Hit (listCache)"| Interceptors
  Service -->|"Cache Miss / listMessages: Promise.all"| Repo
  Repo -->|"find().sort().limit(limit+1).lean()"| DB
  DB --> Repo --> Service --> Interceptors
  Interceptors -->|"JSON CursorPage + _links (hoặc XML)"| Http
```

### 2.2. Sơ đồ Tuần tự (Sequence Diagram: Mount Sync, Cache Hit/Miss & Session Switch)

```mermaid
sequenceDiagram
  autonumber
  actor User as "Người dùng"
  participant UI as "ChatPanel / HistoryDrawer"
  participant Store as "useSessionStore"
  participant Svc as "chatSessionService"
  participant Dexie as "IndexedDB (RagChatDB)"
  participant Http as "http.ts (fetchAllPages / apiFetch)"
  participant Guard as "JwtAuthGuard"
  participant Ctrl as "ChatSessionsController"
  participant Cache as "ChatSessionsService (listCache)"
  participant Repo as "Repositories (.lean)"
  participant Mongo as "MongoDB"
  participant Rag as "useRag"

  User->>UI: "Mở app hoặc thay đổi auth"
  UI->>Store: "initSessions() [useEffect]"
  Store->>Store: "Kiểm tra guard initializedFor === identity"
  Store->>Svc: "listLocalChatSessions()"
  Svc->>Dexie: "chatDb.sessions.orderBy('last_message_at').reverse().toArray()"
  Dexie-->>Store: "LocalChatSession[] (hiển thị tức thì)"

  Store->>Svc: "checkHealth() -> publicFetch('/health', timeout 2000ms)"
  alt "Backend Online & Đã đăng nhập (!USE_MOCK)"
    Store->>Svc: "syncRemoteSessions(userId)"
    Svc->>Http: "fetchAllPages('/chat-sessions', limit=50, maxPages=10)"
    loop "Cho tới khi next_cursor === null hoặc đủ 10 trang"
      Http->>Guard: "GET /api/v1/chat-sessions?limit=50&cursor=..."
      Guard->>Ctrl: "Xác thực Bearer JWT & token_version -> req.user"
      Ctrl->>Cache: "list(me.id, query)"
      alt "listCache Hit (khóa userId:userGen:query)"
        Cache-->>Ctrl: "SessionPageDto từ LRUCache"
      else "listCache Miss"
        Cache->>Repo: "findPageLean(filter, mongoSort, limit + 1)"
        Repo->>Mongo: "find(deleted_at: null, message_count > 0).limit(51).lean()"
        Mongo-->>Repo: "LeanChatSession[]"
        Repo-->>Cache: "rows"
        Cache->>Cache: "toCursorPage() & listCache.set(cacheKey, result)"
        Cache-->>Ctrl: "SessionPageDto"
      end
      Ctrl-->>Http: "200 OK CursorPage + _links"
    end
    Svc->>Dexie: "So sánh updated_at -> chatDb.sessions.put(...) nếu mới hơn"
    Svc-->>Store: "listLocalSessions() đã hợp nhất"
  else "Backend Offline hoặc Mạng lỗi"
    Store->>Store: "isOffline = true, giữ nguyên sessions từ Dexie"
  end

  User->>UI: "Chọn 1 phiên trong HistoryDrawer hoặc auto-rehydrate"
  UI->>Store: "switchSession(sessionId)"
  Store->>Svc: "loadSessionMessages(sessionId, isOffline, hasToken)"
  Svc->>Dexie: "chatDb.messages.where('sessionId').equals(sessionId).sortBy('created_at')"
  alt "Dexie đã có tin nhắn (localMsgs.length > 0)"
    Dexie-->>Svc: "LocalChatMessage[]"
    Svc-->>Store: "localToChatMessage[]"
  else "Dexie rỗng & Online & Có Token"
    Svc->>Http: "fetchAllPages('/chat-sessions/:id/messages', limit=100, maxPages=20)"
    Http->>Guard: "GET /api/v1/chat-sessions/:id/messages?limit=100"
    Guard->>Ctrl: "listMessages(me.id, id, query)"
    Ctrl->>Cache: "listMessages(me.id, id, query)"
    par "1 RTT song song qua Cloudflare Tunnel"
      Cache->>Repo: "findOwnedLean(userId, id)"
      and
      Cache->>Repo: "findPageLean({session_id: id, deleted_at: null}, sort, limit + 1)"
    end
    Repo->>Mongo: "2 truy vấn song song .lean()"
    Mongo-->>Cache: "[session | null, rows]"
    Cache-->>Http: "200 OK MessagePageDto (hoặc 404 Not Found)"
    Svc->>Dexie: "remoteToLocalMessage() -> chatDb.messages.put()"
    Svc-->>Store: "ChatMessage[]"
  else "Dexie rỗng & Offline"
    Svc-->>Store: "[] (mảng rỗng an toàn)"
  end
  Store->>Rag: "loadMessages(messages) (hoặc clearMessages nếu rỗng)"
```

---

## 3. Phân rã Chi tiết 4 Giai đoạn Request-Response

### Giai đoạn 1 — Chiều đi Frontend: Client → FE → HTTP Request

Giai đoạn này bao gồm hai nhánh kích hoạt chính: **(A) Đồng bộ danh sách phiên khi mount/đăng nhập** (`GET /api/v1/chat-sessions`) và **(B) Chuyển phiên & tải tin nhắn lười** (`GET /api/v1/chat-sessions/:id/messages`).

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1.1 | UI Component | [`ChatPanel`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L14-L30) / [`HistoryDrawer`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L24-L100) | Mount component, thay đổi `isAuthenticated`, hoặc click chọn một phiên `session.id` | `ChatPanel` chạy `useEffect` gọi `initSessions()`. Khi người dùng click một mục trong `HistoryDrawer` (và không ở trạng thái `isConfirming`), UI gọi `switchSession(session.id)`. | Gọi hàm từ Selector Hook tương ứng |
| 1.2 | Selector Hook | [`useChatPanelState`](../../../frontend_rag/src/features/chat/hooks/useChatWorkspace.ts#L34-L52) & [`useHistoryDrawerState`](../../../frontend_rag/src/features/chat/hooks/useChatWorkspace.ts#L61-L79) | State từ `useSessionStore` | Tách biệt các selector nhỏ (`sessions`, `currentSessionId`, `isOffline`, `initSessions`, `switchSession`) để tránh re-render dư thừa khi token streaming chạy trong `useRag`. | Trả về các action `init`, `switchSession` |
| 1.3 | State Store (Init) | [`useSessionStore.init`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L75-L118) | `auth.accessToken`, `auth.user?.id` | Kiểm tra guard `initializedFor === identity` ([`sessionStore.ts#L24,L77-L79`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L24)) để không reset hội thoại khi responsive layout mount lại `ChatPanel`. Đọc cache cục bộ qua `listLocalChatSessions()`, đặt `sessions` ngay lập tức, rồi kiểm tra `checkBackendHealth()`. Nếu `isAlive && token && !USE_MOCK`, gọi `syncRemoteSessions(user?.id ?? null)`. | Danh sách phiên cục bộ hiển thị tức thì + kích hoạt đồng bộ nền |
| 1.4 | State Store (Switch) | [`useSessionStore.switchSession`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L129-L146) | `sessionId: string` | Nếu `sessionId === currentSessionId && isStreaming()` thì chỉ đóng drawer và thoát ngay (bảo vệ stream đang chạy, vì state trên RAM mới hơn Dexie). Ngược lại, cập nhật `currentSessionId`, đóng drawer và gọi `loadSessionMessages(sessionId, { isOffline, hasToken })`. | Yêu cầu danh sách `ChatMessage[]` của phiên |
| 1.5 | Service & IndexedDB | [`chatSessionService`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L33-L78) & [`chatDb`](../../../frontend_rag/src/features/chat/db/chatDb.ts#L30-L46) | `userId` hoặc `sessionId` | **Nhánh A (`syncRemoteSessions`)**: gọi `fetchAllRemoteSessions(50, 10, syncRequest)` với `AbortSignal.timeout(4000)`. **Nhánh B (`loadSessionMessages`)**: truy vấn `chatDb.messages.where('sessionId').equals(sessionId).sortBy('created_at')`; nếu có kết quả (`length > 0`) thì map và trả về ngay; chỉ khi mảng rỗng và `hasToken && !USE_MOCK && !isOffline` mới gọi `fetchAllRemoteMessages(sessionId, 100, 20, syncRequest)`. | Kích hoạt gọi API từ xa hoặc trả thẳng dữ liệu Dexie |
| 1.6 | API Wrapper | [`chatApi.ts`](../../../frontend_rag/src/features/chat/api/chatApi.ts#L39-L80) | `limit`, `maxPages`, `initFactory` (`syncRequest`), `sessionId` | `checkBackendHealthApi` dùng `publicFetch('/health', { signal })` (timeout `2000ms`). `fetchAllRemoteSessions` gọi `fetchAllPages('/chat-sessions', { limit: '50' }, 10, initFactory)`. `fetchAllRemoteMessages` gọi `fetchAllPages('/chat-sessions/' + encodeURIComponent(sessionId) + '/messages', { limit: '100' }, 20, initFactory)`. | Gọi vòng lặp phân trang `fetchAllPages` |
| 1.7 | HTTP Client | [`fetchAllPages`](../../../frontend_rag/src/shared/api/http.ts#L37-L55) & [`apiFetch`](../../../frontend_rag/src/shared/api/http.ts#L23-L34) | `path`, `params`, `maxPages`, `accessToken` | `fetchAllPages` lặp tối đa `maxPages`, nối `cursor` vào `URLSearchParams` và gọi `apiFetch`. `apiFetch` ghép `API_BASE` (`/api/v1` theo [`config.ts#L1`](../../../frontend_rag/src/shared/api/config.ts#L1)), gắn `Authorization: Bearer <token>`. Nếu nhận `401`, tự động gọi `rotateTokens(token)` một lần và gửi lại request với token mới. | HTTP GET Request gửi tới Backend |

#### HTTP Wire Request Examples (Frontend → Backend)

**Request A — Lấy trang phiên hội thoại (khi đồng bộ từ xa):**
```http
GET /api/v1/chat-sessions?limit=50&cursor=eyJzIjoibGFzdF9tZXNzYWdlX2F0OmRlc2MiLCJ2IjpbIjIwMjYtMTEtMjZUMDM6MDA6MDAuMDAwWiJdLCJpZCI6IjAxOTM2NmNhLTdmMDAtNzAwMC04MDAwLTAwMDAwMDAwMDAwMSJ9 HTTP/1.1
Host: localhost:5173
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/json
```

**Request B — Lazy-load tin nhắn khi mở phiên chưa có trong IndexedDB:**
```http
GET /api/v1/chat-sessions/019366ca-7f00-7000-8000-000000000001/messages?limit=100 HTTP/1.1
Host: localhost:5173
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/json
```

---

### Giai đoạn 2 — Chiều vào Backend: HTTP Request → BE → Repository → Database

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 2.1 | Global Middlewares | [`configureApp`](../../../backend_rag/src/app.setup.ts#L40-L85) | Express `Request` | Cấu hình `trust proxy`, gắn `X-Request-Id` ([`requestIdMiddleware`](../../../backend_rag/src/app.setup.ts#L52)), thêm security headers (`securityHeaders`), thu thập Prometheus metrics (`metrics.middleware`), kiểm tra prefix `/api/v1`, CORS, và kiểm tra `Accept` header qua `ContentNegotiationInterceptor` (trả `406 Not Acceptable` nếu client không chấp nhận JSON/XML/SSE). | Request sạch đi qua chuỗi NestJS |
| 2.2 | Auth Guard | [`JwtAuthGuard.canActivate`](../../../backend_rag/src/modules/auth/guards/jwt-auth.guard.ts#L28-L46) | Header `Authorization` | `bearerToken(req)` tách chuỗi `Bearer <token>` (sai định dạng hoặc thiếu -> `401`). `tokens.verifyAccess(token)` giải mã JWT; `auth.getActiveUser(claims.id)` kiểm tra tài khoản còn tồn tại, `is_active === true`, và khớp `user.token_version === claims.tokenVersion`. Gắn `req.user = { id, roles, email }`. | `req.user` (`AuthUser`) được gắn vào context |
| 2.3 | Validation Pipe | [`createValidationPipe`](../../../backend_rag/src/common/http/validation.ts#L18-L26) & [`ListSessionsQueryDto` / `ListMessagesQueryDto`](../../../backend_rag/src/modules/chat-sessions/dto/chat-session.dto.ts#L17-L67) | Query params (`limit`, `cursor`, `sort`, `q`, `ids`, `created_at`, `last_message_at`, `message_count`) | `ValidationPipe` (`whitelist: true, forbidNonWhitelisted: true, transform: true`) ép kiểu `limit` thành integer (`1..50` cho sessions mặc định `10`; `1..100` cho messages mặc định `50`), kiểm tra độ dài `cursor` (tối đa 512), `sort` (tối đa 120), `q` (tối đa 120), từ chối tham số lạ với `400 Bad Request`. | DTO hợp lệ truyền vào Controller |
| 2.4 | Controller | [`ChatSessionsController`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.controller.ts#L38-L78) | `@Query() q`, `@Param('id') id`, `@CurrentUser() me` | Route `@Get()` (`list`) gọi `this.sessions.list(me.id, q)`. Route `@Get(':id/messages')` (`listMessages`) gọi `this.sessions.listMessages(me.id, id, q)`. Route `@Get(':id')` (`get`) gọi `this.sessions.get(me.id, id)`. | Ủy quyền xử lý cho `ChatSessionsService` |
| 2.5 | Service (Cache & Query Builder) | [`ChatSessionsService.list`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L114-L165) & [`pagination.ts`](../../../backend_rag/src/common/pagination/pagination.ts#L132-L314) | `userId`, `ListSessionsQueryDto` | **Kiểm tra `listCache`**: tạo khóa `${userId}:${gen}:${JSON.stringify([...])}` từ `userGen.get(userId) ?? 0`. Nếu cache hit, trả về ngay không chạm MongoDB. Nếu cache miss: `parseSort` (mặc định `last_message_at:desc`), `parseFilters` (`created_at`, `last_message_at`, `message_count`), `keysetFilter` giải mã `cursor` và kiểm tra khớp `sortKey(terms)` + `isUuid(c.id)`. Gộp điều kiện cơ sở `{ user_id: ObjectId(userId), deleted_at: null, message_count: { $gt: 0 } }`. Nếu có `q`, áp dụng `{ $regex: escapeRegex(trimmedQ), $options: 'i' }`. Nếu có `ids`, lọc `_id: { $in: parseIds(..., 50, isUuid) }`. | Truy vấn `findPageLean(filter, mongoSort(terms), limit + 1)` |
| 2.6 | Service (Messages & Singleflight) | [`ChatSessionsService.listMessages`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L168-L184) & [`get`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L186-L213) | `userId`, `id`, `ListMessagesQueryDto` | **`listMessages`**: `parseSort` (mặc định `created_at:asc`), dựng `filter` với `{ session_id: id, deleted_at: null }` và `keysetFilter` nếu có `cursor`. Gọi `Promise.all([this.findOwned(id, userId), this.messages.findPageLean(filter, mongoSort(terms), limit + 1)])`. **`get` (Singleflight)**: kiểm tra `isUuid(id)`, tra `sessionDetailCache`, nếu miss thì dùng `inflightDetail` (`Map<string, Promise<SessionDto>>`) để gộp các request đồng thời cho cùng `userId:id` thành 1 lần gọi `loadDetail`. | Thực thi song song 2 truy vấn MongoDB (1 RTT qua tunnel) |
| 2.7 | Repository & Database | [`ChatSessionsRepository`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts#L33-L50) & [`ChatMessagesRepository`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-messages.repository.ts#L29-L35) | `filter`, `mongoSort(terms)`, `limit + 1` | `findPageLean` thực thi `this.model.find(filter).sort(sort).limit(limit + 1).lean()`, tận dụng compound index `{ user_id: 1, deleted_at: 1, last_message_at: -1, _id: -1 }` hoặc `{ session_id: 1, deleted_at: 1, created_at: -1, _id: -1 }`. `findOwnedLean` lọc theo `ownedSessionFilter(userId, id)` (`{ _id: id, user_id: ObjectId(userId), deleted_at: null }`). | `LeanChatSession[]` hoặc `[LeanChatSession \| null, LeanChatMessage[]]` |

---

### Giai đoạn 3 — Chiều về Backend: Database → Repository → Service → Controller → Interceptor → HTTP Response

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 3.1 | Database → Repository | [`ChatSessionsRepository`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts#L33-L50) & [`ChatMessagesRepository`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-messages.repository.ts#L29-L35) | Con trỏ kết quả từ MongoDB | `.lean()` trả về plain JavaScript object (bỏ qua Mongoose Document hydration để tối ưu CPU/RAM dưới tải cao k6). | Mảng `LeanChatSession[]` hoặc `LeanChatMessage[]` (tối đa `limit + 1` phần tử) |
| 3.2 | Service (Ownership & Mapping) | [`ChatSessionsService.findOwned`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L412-L419), [`toCursorPage`](../../../backend_rag/src/common/pagination/pagination.ts#L217-L242), [`toSummary`](../../../backend_rag/src/modules/chat-sessions/chat-session.mapper.ts#L33-L45), [`toMessage`](../../../backend_rag/src/modules/chat-sessions/chat-session.mapper.ts#L47-L62) | `rows`, `limit`, `terms` | Trong `listMessages`, `findOwned` kiểm tra `s !== null`; nếu không tìm thấy (sai chủ sở hữu, đã soft-delete, hoặc `id` không phải UUID) ném `NotFoundException('session not found')` (`404`). `toCursorPage` kiểm tra `hasMore = rows.length > limit`, cắt `rows.slice(0, limit)`, map qua `toSummary` hoặc `toMessage`, và nếu `hasMore` thì `encodeCursor({ s, v, id })` từ bản ghi cuối trang. | `SessionPageDto` hoặc `MessagePageDto` |
| 3.3 | Service (Cache Write) | [`ChatSessionsService.list`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L161-L164) | `cacheKey`, `result` (`SessionPageDto`) | Nếu `cacheEnabled` (ngoài môi trường Jest), lưu `this.listCache.set(cacheKey, result)`. Đối tượng lưu trong cache là bản DTO thuần (chưa gắn `_links`). | `SessionPageDto` trả về Controller |
| 3.4 | Controller → Interceptors | [`LinksInterceptor`](../../../backend_rag/src/common/http/hateoas.ts#L157-L177) & [`ContentNegotiationInterceptor`](../../../backend_rag/src/common/http/xml.ts#L109-L127) | `SessionPageDto` / `MessagePageDto` | `LinksInterceptor` (chạy trước khi serialize body, theo thứ tự đăng ký ở [`app.setup.ts#L77-L81`](../../../backend_rag/src/app.setup.ts#L77-L81)) gọi [`addLinks`](../../../backend_rag/src/common/http/hateoas.ts#L133-L154) tạo **bản sao nông** (`{ ...body }`) để không làm biến đổi object đang chia sẻ trong `listCache`: gắn `_links.self`, `_links.next` (nếu `next_cursor` khác `null`), `_links` cho từng item (`sessionLinks(s.id)`), hoặc `_links.session` trên `MessagePageDto`. Sau đó `ContentNegotiationInterceptor` giữ JSON mặc định hoặc chuyển sang `application/xml` qua `toXml(body)` nếu header `Accept` ưu tiên XML. | JSON (hoặc XML) Response hoàn chỉnh gửi ra mạng |

#### HTTP Wire Response Examples (Backend → Frontend)

**Response A — `GET /api/v1/chat-sessions?limit=50` (200 OK):**
```json
{
  "items": [
    {
      "id": "019366ca-7f00-7000-8000-000000000001",
      "title": "Transformer attention scaling laws",
      "message_count": 4,
      "last_message_at": "2026-11-26T03:00:00.000Z",
      "created_at": "2026-11-26T02:55:00.000Z",
      "updated_at": "2026-11-26T03:00:00.000Z",
      "_links": {
        "self": { "href": "/api/v1/chat-sessions/019366ca-7f00-7000-8000-000000000001" },
        "messages": { "href": "/api/v1/chat-sessions/019366ca-7f00-7000-8000-000000000001/messages" },
        "rename": { "href": "/api/v1/chat-sessions/019366ca-7f00-7000-8000-000000000001", "method": "PATCH" },
        "delete": { "href": "/api/v1/chat-sessions/019366ca-7f00-7000-8000-000000000001", "method": "DELETE" },
        "chat": { "href": "/api/v1/rag/chat", "method": "POST" }
      }
    }
  ],
  "next_cursor": "eyJzIjoibGFzdF9tZXNzYWdlX2F0OmRlc2MiLCJ2IjpbIjIwMjYtMTEtMjZUMDM6MDA6MDAuMDAwWiJdLCJpZCI6IjAxOTM2NmNhLTdmMDAtNzAwMC04MDAwLTAwMDAwMDAwMDAwMSJ9",
  "has_more": true,
  "_links": {
    "self": { "href": "/api/v1/chat-sessions?limit=50" },
    "next": { "href": "/api/v1/chat-sessions?limit=50&cursor=eyJzIjoibGFzdF9tZXNzYWdlX2F0OmRlc2MiLCJ2IjpbIjIwMjYtMTEtMjZUMDM6MDA6MDAuMDAwWiJdLCJpZCI6IjAxOTM2NmNhLTdmMDAtNzAwMC04MDAwLTAwMDAwMDAwMDAwMSJ9" }
  }
}
```

**Response B — `GET /api/v1/chat-sessions/:id/messages?limit=100` (200 OK):**
```json
{
  "items": [
    {
      "id": "019366ca-7f00-7000-8000-000000000010",
      "role": "user",
      "content": "Tóm tắt cơ chế FlashAttention-2",
      "created_at": "2026-11-26T02:55:00.000Z",
      "updated_at": "2026-11-26T02:55:00.000Z"
    },
    {
      "id": "019366ca-7f00-7000-8000-000000000011",
      "role": "assistant",
      "content": "FlashAttention-2 tối ưu hóa thứ tự vòng lặp...",
      "citations": ["2307.08691"],
      "grounded_chunks": [],
      "timings": { "embed_ms": 18, "retrieve_ms": 42, "llm_ms": 620, "total_ms": 680 },
      "created_at": "2026-11-26T02:55:01.000Z",
      "updated_at": "2026-11-26T02:55:01.000Z"
    }
  ],
  "next_cursor": null,
  "has_more": false,
  "_links": {
    "self": { "href": "/api/v1/chat-sessions/019366ca-7f00-7000-8000-000000000001/messages?limit=100" },
    "session": { "href": "/api/v1/chat-sessions/019366ca-7f00-7000-8000-000000000001" }
  }
}
```

---

### Giai đoạn 4 — Chiều về Frontend: HTTP Response → FE → Client

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 4.1 | HTTP Client | [`fetchAllPages`](../../../frontend_rag/src/shared/api/http.ts#L37-L55) | `Response` từng trang (`CursorPage<T>`) | Kiểm tra `res.ok` (nếu lỗi ném `Error`), parse JSON, gộp `data.items` vào mảng tổng, cập nhật `cursor = data.next_cursor`. Lặp tiếp tới khi `!cursor` hoặc chạm trần `maxPages` (`10` trang = 500 sessions; `20` trang = 2000 messages). | Mảng phẳng toàn bộ `items` từ Backend |
| 4.2 | Service & Mapping | [`syncRemoteSessions`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L38-L56), [`loadSessionMessages`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L59-L78), [`messageMapping.ts`](../../../frontend_rag/src/features/chat/lib/messageMapping.ts#L17-L45) | `remote` sessions hoặc `RemoteMessage[]` | **Nhánh A (`syncRemoteSessions`)**: duyệt từng `item`, đọc `chatDb.sessions.get(item.id)`; chỉ ghi đè `chatDb.sessions.put({ ..., synced: true })` khi chưa có hoặc `new Date(item.updated_at) > new Date(existing.updated_at)`, sau đó trả `listLocalSessions()` sắp theo `last_message_at` giảm dần. **Nhánh B (`loadSessionMessages`)**: chuyển đổi qua `remoteToLocalMessage(m, sessionId)` (map `citations` thành `citationChecks` trạng thái `'verified'`), lưu xuống `chatDb.messages.put(m)`, và map sang `ChatMessage[]` qua `localToChatMessage(m, index)` (gán `runId = index + 1`). | `LocalChatSession[]` đã hợp nhất hoặc `ChatMessage[]` đã chuẩn hóa |
| 4.3 | State Store | [`useSessionStore.init`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L92-L117) & [`switchSession`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L137-L145) | `merged` sessions hoặc `messages: ChatMessage[]` | Trong `init()`: cập nhật `set({ sessions: merged })`, chọn `targetId` (ưu tiên `currentSessionId` đã persist trong `localStorage` qua key `rag-session-storage` nếu còn tồn tại, ngược lại lấy `sessions[0].id`), rồi gọi `switchSession(targetId)` hoặc `startNewChat()`. Trong `switchSession()`: nếu `messages.length > 0`, gọi `useRag.getState().loadMessages(messages)`; ngược lại gọi `clearMessages()`. | State `sessions`, `currentSessionId` và `useRag.messages` được đồng bộ |
| 4.4 | RAG Store Hydration | [`useRag.loadMessages`](../../../frontend_rag/src/features/chat/store/ragStore.ts#L127-L137) | `messages: ChatMessage[]` | Gọi `cancelRun('loadMessages (session switch)')` để hủy mọi SSE stream cũ còn treo, tính `maxRunId`, cập nhật `messages`, đặt `phase: 'done'`, `visualPhase: 'done'`, `current: emptyRun()`, và `runId: maxRunId + 1` để lượt hỏi tiếp theo nối tiếp số thứ tự chính xác. | Giao diện khung chat sẵn sàng hiển thị và tiếp tục hội thoại |
| 4.5 | UI Render | [`ChatPanel`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L58-L73) & [`HistoryDrawer`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L75-L152) | `sessions`, `currentSessionId`, `isOffline`, `messages` | `ChatPanel` hiển thị tiêu đề `activeSession?.title` và badge `Ngoại tuyến` (nếu `isOffline && !USE_MOCK`); `HistoryDrawer` render danh sách phiên theo thời gian tương đối (`formatRelativeTime`) và `MessageList` hiển thị toàn bộ hội thoại đã hydrate. | Trải nghiệm người dùng hoàn chỉnh (online lẫn offline) |

---

## 4. Đường dẫn Lỗi & Các Bất biến Hệ thống (Error Paths & System Invariants)

1. **Chống reset hội thoại khi đổi kích thước màn hình (`initializedFor` guard)**:
   - Khi thu hẹp viewport (ví dụ mở DevTools), `AppShell` hoán đổi giữa dock desktop và bottom sheet mobile làm [`ChatPanel`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L27-L29) unmount/mount lại. Biến module `initializedFor` trong [`sessionStore.ts#L19-L24,L76-L79`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L19-L24) ghi nhớ danh tính phiên (`user.id`, `'signed-in'`, hoặc `'anonymous'`) để bỏ qua các lần `init()` lặp lại, ngăn việc tải lại phiên làm đứt đoạn SSE stream đang chạy.
   - Tương tự, [`switchSession`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L131-L134) kiểm tra `sessionId === currentSessionId && isStreaming()` để giữ nguyên state trên RAM thay vì ghi đè bằng bản cũ từ Dexie.
2. **Luôn trả `404 Not Found` (tuyệt đối không trả `403 Forbidden`)**:
   - Mọi truy vấn đơn lẻ tới phiên (`listMessages`, `get`, `rename`, `saveExchange`) đều đi qua [`findOwned`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L412-L419) và [`ownedSessionFilter`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts#L19-L24). Nếu `id` không phải `UUID` hợp lệ (`!isUuid(id)`), thuộc tài khoản khác, hoặc đã soft-delete, hệ thống đều ném `NotFoundException('session not found')` (`404`) để không làm rò rỉ sự tồn tại của tài nguyên người dùng khác.
3. **Lọc bản ghi chưa xóa mềm (`deleted_at: null`) & TTL 3 ngày**:
   - Mọi query trên `chat_sessions` và `chat_messages` bắt buộc kèm `deleted_at: null` ([`chat-sessions.service.ts#L143,L176`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L143), [`chat-sessions.repository.ts#L23`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts#L23)). Bộ lọc danh sách phiên còn yêu cầu `message_count: { $gt: 0 }` ([`chat-sessions.service.ts#L144`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L144)) để ẩn các phiên rỗng. Cả hai collection đều gắn MongoDB TTL index trên `deleted_at` hết hạn sau `3 ngày` (`SESSION_TTL_SECONDS`, [`chat-session.schema.ts#L42-L45`](../../../backend_rag/src/modules/chat-sessions/schemas/chat-session.schema.ts#L42-L45)).
4. **Chống ReDoS khi tìm kiếm tiêu đề (`escapeRegex(q)`)**:
   - Tham số tìm kiếm `q` ([`chat-sessions.service.ts#L148-L149`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L148-L149)) và toán tử `match:` ([`pagination.ts#L257-L261`](../../../backend_rag/src/common/pagination/pagination.ts#L257-L261)) đều đi qua [`escapeRegex`](../../../backend_rag/src/common/security/sanitize.ts#L7-L9) trước khi truyền vào `$regex` của MongoDB, vô hiệu hóa các ký tự meta biểu thức chính quy gây treo CPU.
5. **Khóa cache thế hệ theo từng người dùng (`userGen` + `listCache`) & tính bất biến (`immutability`)**:
   - [`ChatSessionsService`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L81-L112) dùng `LRUCache` (`listCache`, tối đa `10_000` khóa, TTL `300_000ms`, `updateAgeOnGet: true`) với khóa chứa `userGen.get(userId)`. Mọi thao tác ghi (`saveExchange`, `rename`, `remove`, `removeMany`) gọi `this.invalidate(userId, ...)` để tăng `userGen` ngay lập tức (O(1)), khiến toàn bộ các trang danh sách cũ của người dùng đó hết hiệu lực tức thì.
   - [`LinksInterceptor` / `addLinks`](../../../backend_rag/src/common/http/hateoas.ts#L132-L154) luôn tạo bản sao nông của body và mảng `items` trước khi gắn `_links` nhằm không làm bẩn dữ liệu dùng chung trong `listCache`.
6. **Chống N+1 qua Cloudflare Tunnel (`Promise.all` & `.lean()`)**:
   - `listMessages` thực thi song song `Promise.all([this.findOwned(id, userId), this.messages.findPageLean(...)])` ([`chat-sessions.service.ts#L179-L182`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L179-L182)), và `loadDetail` thực thi song song `Promise.all([findOwnedLean, findRecentForSessionLean])` ([`chat-sessions.service.ts#L207-L210`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L207-L210)), giữ tổng độ trễ mạng ở mức 1 RTT (~15–25ms qua đường hầm `cloudflared`).
   - Toàn bộ truy vấn đọc đều gọi `.lean()` và ràng buộc con trỏ `keysetFilter` theo đúng `sort` (nếu client đổi `sort` nhưng tái sử dụng `cursor` cũ -> `400 invalid cursor` tại [`pagination.ts#L190-L192`](../../../backend_rag/src/common/pagination/pagination.ts#L190-L192)).
7. **Tính liên tục khi ngoại tuyến (`isOffline` Continuity)**:
   - `checkHealth()` có timeout cứng `2000ms`, các request đồng bộ dùng `SYNC_REQUEST_TIMEOUT_MS = 4000ms` ([`chatSessionService.ts#L20-L31`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L20-L31)). Nếu Backend sập hoặc rớt mạng, `syncRemoteSessions` ném lỗi được `init()` bắt và log cảnh báo (`continuing offline`), giữ nguyên `sessions` đã nạp từ `RagChatDB`, còn `loadSessionMessages` âm thầm trả `[]` nếu IndexedDB chưa có tin nhắn — bảo đảm ứng dụng không bao giờ crash khi mất kết nối.
