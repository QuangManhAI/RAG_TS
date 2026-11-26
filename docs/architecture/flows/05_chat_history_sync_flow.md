# Luồng 05 — Đồng bộ Lịch sử Hội thoại Offline-First & Phân trang UUIDv7 Cursor (GET /api/v1/chat-sessions)

- **Motivation/Background**: Trong nền tảng `RAG_TS`, người dùng cần xem lại toàn bộ các phiên trò chuyện khoa học (`chat_sessions`) và lịch sử tin nhắn (`chat_messages`) tức thì ngay cả khi máy chủ ngoại tuyến hoặc đang chạy qua đường hầm mạng (`Cloudflare Access TCP Tunnel` tới MongoDB từ xa tốn ~15–25ms mỗi round-trip). Đồng thời, cơ chế phân trang phải tránh lỗi bỏ sót/trùng lặp dữ liệu khi có tin nhắn mới xen vào và bảo vệ tuyệt đối quyền riêng tư giữa các tài khoản bằng khóa chính `UUIDv7` và kiểm tra quyền sở hữu tầng ứng dụng.
- **Purpose**: Tài liệu hóa chi tiết luồng request-response đầu-cuối (end-to-end) cho kiến trúc đồng bộ lịch sử hội thoại Offline-First (`Cache-First` với `Dexie.js` IndexedDB) kết hợp phân trang `UUIDv7` Keyset Cursor qua hai endpoint chính `GET /api/v1/chat-sessions` và `GET /api/v1/chat-sessions/:id/messages` (cùng nhánh `Singleflight` của `GET /api/v1/chat-sessions/:id`).
- **Overview Pipeline**: Mount/Auth (`ChatPanel`) → `useSessionStore.init()` đọc `Dexie.js` tức thì → `GET /api/v1/health` → `fetchAllPages` phân trang `GET /api/v1/chat-sessions` → Backend `LRUCache` theo `userGen` & Keyset Query MongoDB `.lean()` → Hợp nhất Last-Write-Wins vào `RagChatDB` → Chọn phiên (`switchSession`) tải tin nhắn từ IndexedDB hoặc lazy-fetch `GET /api/v1/chat-sessions/:id/messages` qua `Promise.all` → Hydrate `useRag.loadMessages()`.
- **Detailed Plan**: Mục 1 tổng quan kiến trúc Offline-First & phân trang Keyset; Mục 2 sơ đồ luồng thành phần (`flowchart TB`) và tuần tự (`sequenceDiagram`); Mục 3 phân rã chi tiết 4 giai đoạn kèm HTTP Wire Examples; Mục 4 đường dẫn lỗi và các bất biến hệ thống (`System Invariants`).
- **References**: [ChatPanel.tsx](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx), [HistoryDrawer.tsx](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx), [useChatWorkspace.ts](../../../frontend_rag/src/features/chat/hooks/useChatWorkspace.ts), [sessionStore.ts](../../../frontend_rag/src/features/chat/store/sessionStore.ts), [ragStore.ts](../../../frontend_rag/src/features/chat/store/ragStore.ts), [chatSessionService.ts](../../../frontend_rag/src/features/chat/services/chatSessionService.ts), [chatApi.ts](../../../frontend_rag/src/features/chat/api/chatApi.ts), [chatDb.ts](../../../frontend_rag/src/features/chat/db/chatDb.ts), [messageMapping.ts](../../../frontend_rag/src/features/chat/lib/messageMapping.ts), [http.ts](../../../frontend_rag/src/shared/api/http.ts), [config.ts](../../../frontend_rag/src/shared/api/config.ts), [app.setup.ts](../../../backend_rag/src/app.setup.ts), [jwt-auth.guard.ts](../../../backend_rag/src/modules/auth/guards/jwt-auth.guard.ts), [pagination.ts](../../../backend_rag/src/common/pagination/pagination.ts), [sanitize.ts](../../../backend_rag/src/common/security/sanitize.ts), [hateoas.ts](../../../backend_rag/src/common/http/hateoas.ts), [validation.ts](../../../backend_rag/src/common/http/validation.ts), [xml.ts](../../../backend_rag/src/common/http/xml.ts), [chat-sessions.controller.ts](../../../backend_rag/src/modules/chat-sessions/chat-sessions.controller.ts), [chat-sessions.service.ts](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts), [chat-session.mapper.ts](../../../backend_rag/src/modules/chat-sessions/chat-session.mapper.ts), [chat-session.dto.ts](../../../backend_rag/src/modules/chat-sessions/dto/chat-session.dto.ts), [chat-sessions.constants.ts](../../../backend_rag/src/modules/chat-sessions/chat-sessions.constants.ts), [chat-sessions.repository.ts](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts), [chat-messages.repository.ts](../../../backend_rag/src/modules/chat-sessions/repositories/chat-messages.repository.ts), [chat-session.schema.ts](../../../backend_rag/src/modules/chat-sessions/schemas/chat-session.schema.ts), [chat-message.schema.ts](../../../backend_rag/src/modules/chat-sessions/schemas/chat-message.schema.ts).
- **Created**: 2026-11-26T10:12:00+07:00
- **Last Updated**: 2026-11-26T12:00:00+07:00

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

### 2.1. Sơ đồ Luồng Thành phần Chuẩn 26 Bước (`flowchart TB`)

```mermaid
flowchart TB
  subgraph FE_IN["Giai đoạn 1 — Frontend Chiều đi (Client -> HTTP Request)"]
    CLI_IN["1. Client (Mở app / đổi đăng nhập / bấm chọn phiên trong HistoryDrawer)"]
    UI_IN["2. FE Component (ChatPanel / HistoryDrawer)"]
    HK_IN["3. FE Hook & Store (useChatPanelState / useHistoryDrawerState -> useSessionStore)"]
    SVC_IN["4. FE Service (chatSessionService: Cache-First Dexie check + timeout 4s)"]
    DEXIE[("IndexedDB: RagChatDB (sessions, messages)")]
    API_IN["5. FE API (chatApi.fetchAllRemoteSessions / fetchAllRemoteMessages)"]
    HTTP_IN["6. FE HTTP Client (http.ts: fetchAllPages + apiFetch GET /chat-sessions)"]
    CLI_IN --> UI_IN --> HK_IN --> SVC_IN --> API_IN --> HTTP_IN
    SVC_IN <--> DEXIE
  end

  subgraph BE_IN["Giai đoạn 2 — Backend Chiều vào (HTTP Request -> Database)"]
    NODE["7. BE Node.js HTTP Server (main.ts: keepAliveTimeout=65s, headersTimeout=66s)"]
    MW["8. BE NestJS/Express Middleware (configureApp: trust proxy, requestId, helmet, metrics, XML/JSON)"]
    GRD["9. BE Guard (ThrottlerGuard 120/min + JwtAuthGuard token_version)"]
    INT_IN["10. BE Interceptor chiều vào (ContentNegotiation Accept 406 + LinksInterceptor metadata)"]
    PIP["11. BE Pipe (ValidationPipe -> ListSessionsQueryDto / ListMessagesQueryDto)"]
    CTL_IN["12. BE Controller (ChatSessionsController: list, listMessages, get)"]
    BSVC_IN["13. BE Service (ChatSessionsService: listCache userGen, Singleflight, Promise.all)"]
    REPO_IN["14. BE Repository (ChatSessionsRepository & ChatMessagesRepository .lean())"]
    DB[("15. Database (MongoDB: chat_sessions & chat_messages)")]
    HTTP_IN -->|"GET /api/v1/chat-sessions & :id/messages"| NODE --> MW --> GRD --> INT_IN --> PIP --> CTL_IN --> BSVC_IN --> REPO_IN --> DB
  end

  subgraph BE_OUT["Giai đoạn 3 — Backend Chiều về (Database -> HTTP Response)"]
    REPO_OUT["16. Database -> BE Repository (LeanChatSession[] & LeanChatMessage[])"]
    BSVC_OUT["17. BE Service (assertOwned 404, toCursorPage, nạp listCache)"]
    CTL_OUT["18. BE Controller (ChatSessionsController trả SessionPageDto / MessagePageDto)"]
    INT_OUT["19. BE Interceptor chiều ra (LinksInterceptor +_links bản sao nông -> ContentNegotiation JSON/XML)"]
    EXP_OUT["20. BE NestJS/Express gửi response (HTTP 200 OK, X-Request-Id, metrics finalize)"]
    DB --> REPO_OUT --> BSVC_OUT --> CTL_OUT --> INT_OUT --> EXP_OUT
  end

  subgraph FE_OUT["Giai đoạn 4 — Frontend Chiều về (HTTP Response -> Client)"]
    HTTP_OUT["21. FE HTTP Client (apiFetch 401 retry + fetchAllPages kiểm tra res.ok & gom trang)"]
    API_OUT["22. FE API (chatApi trả SessionSummary[] / RemoteMessage[] phẳng)"]
    SVC_OUT["23. FE Service (chatSessionService hợp nhất LWW / remoteToLocalMessage -> lưu RagChatDB)"]
    HK_OUT["24. FE Hook & Store (useSessionStore + useRag.loadMessages -> Selector Hooks)"]
    UI_OUT["25. FE Component (ChatPanel tiêu đề + HistoryDrawer + MessageList re-render)"]
    CLI_OUT["26. Client (Người dùng xem lịch sử chat tức thì online lẫn offline)"]
    EXP_OUT -->|"HTTP 200 CursorPage JSON/XML"| HTTP_OUT --> API_OUT --> SVC_OUT --> HK_OUT --> UI_OUT --> CLI_OUT
  end
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
| 1 | Client | [`ChatPanel`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L14-L30) / [`HistoryDrawer`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L24-L100) | Trình duyệt vừa mở app, vừa đăng nhập xong hoặc người dùng click chọn một phiên | Thao tác mở khung chat hoặc nhấp chuột vào một mục phiên hội thoại trong `HistoryDrawer`. | Sự kiện mount / click trên trình duyệt |
| 2 | FE — Component | [`ChatPanel`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L27-L30) / [`HistoryDrawer`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L83-L100) | Mount `useEffect` hoặc click mục `session.id` | `ChatPanel` chạy `useEffect` kích hoạt `initSessions()`; `HistoryDrawer` kiểm tra `!isConfirming` rồi gọi `switchSession(session.id)`. | Gọi action từ Selector Hook tương ứng |
| 3 | FE — Hook (+ Store) | [`useChatPanelState`](../../../frontend_rag/src/features/chat/hooks/useChatWorkspace.ts#L34-L52), [`useHistoryDrawerState`](../../../frontend_rag/src/features/chat/hooks/useChatWorkspace.ts#L61-L79), [`sessionStore.ts`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L75-L146) | `auth.accessToken`, `auth.user?.id`, `sessionId` | Hooks tách selector tránh re-render dư thừa. `useSessionStore.init` kiểm tra guard `initializedFor`, đọc ngay Dexie để hiển thị 0ms rồi gọi `syncRemoteSessions`; `switchSession` bảo vệ stream đang chạy và gọi `loadSessionMessages`. | Yêu cầu thực thi xuống `chatSessionService` |
| 4 | FE — Service | [`chatSessionService`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L33-L78) & [`chatDb`](../../../frontend_rag/src/features/chat/db/chatDb.ts#L30-L46) | `userId` hoặc `(sessionId, { isOffline, hasToken })` | **Nhánh A (`syncRemoteSessions`)**: kiểm tra `checkHealth()` và gọi `fetchAllRemoteSessions(50, 10, syncRequest)` (timeout `4000ms`). **Nhánh B (`loadSessionMessages`)**: đọc `chatDb.messages.where('sessionId')` trước; chỉ khi cục bộ rỗng và `hasToken && !isOffline && !USE_MOCK` mới gọi `fetchAllRemoteMessages`. | Trả dữ liệu cục bộ ngay hoặc kích hoạt API từ xa |
| 5 | FE — API | [`chatApi.ts`](../../../frontend_rag/src/features/chat/api/chatApi.ts#L39-L80) | `limit`, `maxPages`, `syncRequest`, `sessionId` | `fetchAllRemoteSessions` truyền `'/chat-sessions'`, `{ limit: '50' }`, `maxPages = 10` tới `fetchAllPages`; `fetchAllRemoteMessages` truyền `'/chat-sessions/' + encodeURIComponent(id) + '/messages'`, `{ limit: '100' }`, `maxPages = 20`. | Lời gọi phân trang `fetchAllPages` |
| 6 | FE — HTTP Client (gọi fetch) | [`fetchAllPages`](../../../frontend_rag/src/shared/api/http.ts#L37-L55), [`apiFetch`](../../../frontend_rag/src/shared/api/http.ts#L23-L34), [`config.ts#L1`](../../../frontend_rag/src/shared/api/config.ts#L1) | `path`, `params`, `accessToken` | `fetchAllPages` lặp nối `cursor` vào `URLSearchParams` và gọi `apiFetch`; `apiFetch` ghép `API_BASE` (`/api/v1`), gắn `Authorization: Bearer <token>`, gọi `fetch()` và tự xoay token 1 lần nếu gặp `401`. | HTTP GET Request gửi tới Backend |

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

### Giai đoạn 2 — Chiều vào Backend: HTTP Request → BE → Database

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 7 | BE — Node.js HTTP Server | [`main.ts`](../../../backend_rag/src/main.ts#L1-L26) (`bootstrap`) | HTTP socket cổng `3000` | Node `http.Server` giữ kết nối qua proxy/Cloudflare (`keepAliveTimeout = 65_000`, `headersTimeout = 66_000`), tiếp nhận GET request. | Kết nối chuyển vào Express/NestJS |
| 8 | BE — NestJS/Express Middleware | [`configureApp`](../../../backend_rag/src/app.setup.ts#L40-L85) | Raw Express `Request` | Cấu hình `trust proxy`, `requestIdMiddleware` (`X-Request-Id` UUIDv7 + `AsyncLocalStorage`), `securityHeaders` (Helmet), `PrometheusMetricsRegistry.middleware`, global prefix `/api/v1` và CORS. | `req` gắn `req.id` và ngữ cảnh log |
| 9 | BE — Guard | [`throttler.ts`](../../../backend_rag/src/common/security/throttler.ts#L16-L46) (`ThrottlerGuard`) & [`jwt-auth.guard.ts`](../../../backend_rag/src/modules/auth/guards/jwt-auth.guard.ts#L28-L46) (`JwtAuthGuard`) | `req.ip`, header `Authorization` | `ThrottlerGuard` áp bucket mặc định `120 req/60s/IP`; `JwtAuthGuard` trích `Bearer <token>`, gọi `tokens.verifyAccess(token)` và `auth.getActiveUser(claims.id)` (cache `5s`) kiểm tra `is_active === true` và khớp `token_version === claims.tokenVersion`, gán `req.user`. | `req.user` (`AuthUser`) hoặc `401`/`429` |
| 10 | BE — Interceptor chiều vào | [`xml.ts`](../../../backend_rag/src/common/http/xml.ts#L109-L119) (`ContentNegotiationInterceptor`) & [`hateoas.ts`](../../../backend_rag/src/common/http/hateoas.ts#L157-L166) (`LinksInterceptor`) | Header `Accept`, `@WithLinks` | `ContentNegotiationInterceptor` kiểm tra `req.accepts(OFFERED)` (ném `406 NotAcceptableException` nếu không hợp lệ); `LinksInterceptor` đọc spec `@WithLinks` (`item: sessionLinks` hoặc `resource`). | Request vượt qua tiền kiểm tra |
| 11 | BE — Pipe | [`createValidationPipe`](../../../backend_rag/src/common/http/validation.ts#L18-L26) & [`ListSessionsQueryDto` / `ListMessagesQueryDto`](../../../backend_rag/src/modules/chat-sessions/dto/chat-session.dto.ts#L17-L67) | Query params (`limit`, `cursor`, `sort`, `q`, `ids`, bộ lọc) | `ValidationPipe` (`whitelist: true, forbidNonWhitelisted: true, transform: true`) ép kiểu `limit` (`1..50` sessions; `1..100` messages), kiểm tra `cursor <= 512`, `sort <= 120`, `q <= 120`; từ chối query lạ với `400 Bad Request`. | DTO query hợp lệ truyền vào Controller |
| 12 | BE — Controller | [`ChatSessionsController`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.controller.ts#L43-L78) | `@Query() q`, `@Param('id') id`, `@CurrentUser() me` | `list` gọi `this.sessions.list(me.id, q)`; `listMessages` gọi `this.sessions.listMessages(me.id, id, q)`; `get` gọi `this.sessions.get(me.id, id)`. | Ủy quyền xuống `ChatSessionsService` |
| 13 | BE — Service | [`ChatSessionsService`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L114-L213) & [`pagination.ts`](../../../backend_rag/src/common/pagination/pagination.ts#L132-L314) | `userId`, `id`, query DTO | **`list`**: tra `listCache` khóa `${userId}:${userGen}:${query}`; nếu miss, chạy `parseSort` (`last_message_at:desc`), `parseFilters`, `keysetFilter` giải mã `cursor`, gộp `{ user_id, deleted_at: null, message_count: { $gt: 0 } }` + `escapeRegex(q)`. **`listMessages`**: dựng filter `{ session_id: id, deleted_at: null }` và chuẩn bị `Promise.all` kiểm tra sở hữu + phân trang. **`get`**: dùng `sessionDetailCache` + `inflightDetail` (`Singleflight`). | Lời gọi Repository `.lean()` (hoặc trả Cache Hit) |
| 14 | BE — Repository | [`ChatSessionsRepository`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts#L33-L50) & [`ChatMessagesRepository`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-messages.repository.ts#L29-L35) | `filter`, `mongoSort(terms)`, `limit + 1` | `findPageLean` xây dựng truy vấn `model.find(filter).sort(sort).limit(limit + 1).lean()`. Với `listMessages`, `findOwnedLean` và `messages.findPageLean` chạy song song trong `Promise.all` để giữ tổng trễ ở mức **1 RTT**. | Lệnh đọc Mongoose gửi xuống MongoDB |
| 15 | Database | [`chat-session.schema.ts`](../../../backend_rag/src/modules/chat-sessions/schemas/chat-session.schema.ts#L5-L45) & [`chat-message.schema.ts`](../../../backend_rag/src/modules/chat-sessions/schemas/chat-message.schema.ts#L5-L54) | Truy vấn `.lean()` qua Cloudflare TCP Tunnel | MongoDB sử dụng compound index `{ user_id: 1, deleted_at: 1, last_message_at: -1, _id: -1 }` và `{ session_id: 1, deleted_at: 1, created_at: -1, _id: -1 }` đọc tối đa `limit + 1` bản ghi (UUIDv7 PK). | Con trỏ kết quả plain JS objects |

---

### Giai đoạn 3 — Chiều về Backend: Database → BE → HTTP Response

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 16 | Database → BE — Repository | [`ChatSessionsRepository`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts#L33-L50) & [`ChatMessagesRepository`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-messages.repository.ts#L29-L35) | Kết quả `.lean()` từ MongoDB | Trả plain JS object (`LeanChatSession[]` hoặc `[LeanChatSession \| null, LeanChatMessage[]]`) bỏ qua Mongoose hydration để tối ưu CPU/RAM dưới tải k6 `400 CUs`. | Mảng plain objects về `ChatSessionsService` |
| 17 | BE — Service | [`ChatSessionsService`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L161-L184), [`toCursorPage`](../../../backend_rag/src/common/pagination/pagination.ts#L217-L242), [`chat-session.mapper.ts`](../../../backend_rag/src/modules/chat-sessions/chat-session.mapper.ts#L33-L62) | `rows`, `limit`, `terms` | Kiểm tra sở hữu qua `findOwned` (ném `404 NotFoundException('session not found')` nếu `null` — không bao giờ trả `403`); `toCursorPage` cắt `limit` phần tử, xác định `has_more`, mã hóa `next_cursor`, map sang `SessionSummaryDto` / `MessageDto`; nạp kết quả thuần vào `listCache` nếu `cacheEnabled`. | `SessionPageDto` hoặc `MessagePageDto` |
| 18 | BE — Controller | [`ChatSessionsController`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.controller.ts#L43-L78) | `Promise<SessionPageDto \| MessagePageDto>` | Trả nguyên DTO phân trang về pipeline NestJS với HTTP status `200 OK`. | DTO phân trang đi qua Interceptor chiều ra |
| 19 | BE — Interceptor chiều ra | [`LinksInterceptor`](../../../backend_rag/src/common/http/hateoas.ts#L133-L177) & [`ContentNegotiationInterceptor`](../../../backend_rag/src/common/http/xml.ts#L119-L127) | `SessionPageDto` / `MessagePageDto` | `LinksInterceptor` sao chép nông `{ ...body }` và `items.map(...)` (bảo vệ object chia sẻ trong `listCache`) để gắn `_links.self`, `_links.next`, `_links` từng item. Sau đó `ContentNegotiationInterceptor` giữ JSON hoặc chuyển `application/xml` qua `toXml`. | Payload hoàn chỉnh đã gắn `_links` |
| 20 | BE — NestJS/Express gửi response | [`app.setup.ts`](../../../backend_rag/src/app.setup.ts#L52-L63), [`prometheus-metrics.ts`](../../../backend_rag/src/common/metrics/prometheus-metrics.ts#L57-L76), [`http-exception.filter.ts`](../../../backend_rag/src/common/http/http-exception.filter.ts#L69-L97) | Body JSON/XML hoặc `HttpException` (`400`/`401`/`404`) | Express gửi HTTP `200 OK` kèm header `X-Request-Id` và security headers (hoặc `HttpExceptionFilter` chuẩn hóa lỗi `400`/`401`/`404`); sự kiện `finish` ghi histogram Prometheus. | HTTP Response rời server về Frontend |

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
| 21 | FE — HTTP Client (kiểm tra status & đọc body) | [`fetchAllPages`](../../../frontend_rag/src/shared/api/http.ts#L37-L55) & [`apiFetch`](../../../frontend_rag/src/shared/api/http.ts#L23-L34) | `Response` từng trang từ Backend | `apiFetch` tự xoay refresh token nếu `401`. `fetchAllPages` kiểm tra `res.ok` (nếu lỗi ném `Error`), đọc `res.json()` thành `CursorPage<T>`, gộp `data.items` và lặp theo `next_cursor` tới khi `null` hoặc chạm trần `maxPages` (`10` trang sessions / `20` trang messages). | Mảng phẳng `items: T[]` |
| 22 | FE — API (xử lý dữ liệu) | [`chatApi.ts`](../../../frontend_rag/src/features/chat/api/chatApi.ts#L45-L80) (`fetchAllRemoteSessions`, `fetchAllRemoteMessages`) | `RemoteSessionSummary[]` / `RemoteMessage[]` | Giải quyết `Promise` trả danh sách DTO đã gộp qua nhiều trang từ API adapter về cho tầng service (hoặc lan truyền ngoại lệ timeout/HTTP khi lỗi). | Dữ liệu remote sạch theo kiểu TypeScript |
| 23 | FE — Service | [`syncRemoteSessions`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L38-L56), [`loadSessionMessages`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L59-L78), [`messageMapping.ts`](../../../frontend_rag/src/features/chat/lib/messageMapping.ts#L17-L45) | `remote` sessions hoặc `RemoteMessage[]` | **Nhánh A (`syncRemoteSessions`)**: so sánh `updated_at` kiểu **Last-Write-Wins** rồi ghi đè `chatDb.sessions.put({ ..., synced: true })`, trả `listLocalSessions()`. **Nhánh B (`loadSessionMessages`)**: chuyển qua `remoteToLocalMessage` (map `citations` thành `'verified'`), lưu xuống `chatDb.messages.put()`, map sang `ChatMessage[]`. | `LocalChatSession[]` hoặc `ChatMessage[]` đã cache vào Dexie |
| 24 | FE — Hook (cập nhật state) | [`sessionStore.ts`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L92-L145), [`ragStore.ts`](../../../frontend_rag/src/features/chat/store/ragStore.ts#L127-L137), [`useChatWorkspace.ts`](../../../frontend_rag/src/features/chat/hooks/useChatWorkspace.ts#L34-L79) | `merged` sessions hoặc `messages: ChatMessage[]` | `useSessionStore` cập nhật `sessions`, chọn `targetId` (ưu tiên `currentSessionId` đã lưu `localStorage`), gọi `switchSession(targetId)` → `useRag.loadMessages(messages)` hủy stream cũ và đặt `runId = maxRunId + 1`; các selector Hooks phát state mới. | State `sessions`, `currentSessionId`, `messages` đồng bộ |
| 25 | FE — Component (hiển thị lại) | [`ChatPanel`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L58-L73), [`HistoryDrawer`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L75-L152), [`MessageList`](../../../frontend_rag/src/features/chat/components/MessageList.tsx#L12-L45) | `sessions`, `currentSessionId`, `isOffline`, `messages` | `ChatPanel` hiển thị tiêu đề `activeSession?.title` & badge `Ngoại tuyến` (nếu offline); `HistoryDrawer` sắp xếp danh sách phiên theo `formatRelativeTime`; `MessageList` render toàn bộ bong bóng chat & chip trích dẫn đã hydrate. | DOM giao diện chat & lịch sử cập nhật hoàn chỉnh |
| 26 | Client | [`ChatPanel`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L58-L73) & [`HistoryDrawer`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L75-L152) | Giao diện sau re-render | Người dùng xem lại toàn bộ các phiên và lịch sử hỏi đáp (kể cả khi mất kết nối Backend nhờ IndexedDB) và có thể tiếp tục đặt câu hỏi nối tiếp. | Trải nghiệm Offline-First liền mạch |

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
