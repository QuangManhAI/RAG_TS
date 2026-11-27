# Luồng 06 — Quản lý Phiên Chat: Đổi tên & Xóa mềm Idempotent (`DELETE` & `PATCH /api/v1/chat-sessions/:id`)

- **Motivation/Background**: Người dùng cần xóa hoặc đồng bộ tiêu đề các phiên trò chuyện một cách tức thì trên giao diện ngay cả khi mất kết nối mạng, trong khi Backend phải bảo đảm tính toàn vẹn đa người dùng (không rò rỉ thông tin sở hữu), tính lũy đẳng (`Idempotent`) của phép xóa và vô hiệu hóa bộ nhớ đệm (`Cache Invalidation`) ở độ phức tạp `O(1)` cho danh sách phân trang nhiều chiều.
- **Purpose**: Tài liệu hóa chi tiết luồng Request–Response đầu-cuối cho hai thao tác ghi trên tài nguyên phiên chat: xóa mềm có xác nhận 2 bước (`DELETE /api/v1/chat-sessions/:id`) và tự động đồng bộ tiêu đề sau lượt hỏi đầu tiên (`PATCH /api/v1/chat-sessions/:id`).
- **Overview Pipeline**: Truy vết trực tiếp từ UI `HistoryDrawer` / `useRag` → Zustand Stores (`sessionStore`) → Dexie IndexedDB (`chatDb`) → `chatSessionService` → `chatApi` / `apiFetch` → NestJS Middleware & `JwtAuthGuard` → `ChatSessionsController` → `ChatSessionsService` (`invalidate` `O(1)` qua `userGen`) → `ChatSessionsRepository` & `ChatMessagesRepository` → MongoDB (TTL 3 ngày trên `deleted_at`) → Interceptors & HTTP Response.
- **Detailed Plan**: §1 Tổng quan kiến trúc luồng đột biến phiên chat; §2 Sơ đồ khối kiến trúc (Flowchart); §3 Sơ đồ tuần tự End-to-End (Sequence Diagram); §4 Phân rã 4 giai đoạn Request–Response & Wire Examples; §5 Các nhánh lỗi, ràng buộc Idempotency & Bất biến hệ thống.
- **References**: React 19, Zustand, Dexie.js (IndexedDB), NestJS 11, Mongoose/MongoDB (TTL Index), `lru-cache`, `class-validator`, `AGENTS.md`, `agents/rules/MD_CONVENTION.md`, `docs/api/API_DESIGN.md`.
- **Created**: 2026-11-26T10:12:00+07:00
- **Last Updated**: 2026-11-27T07:40:00+07:00

[STATUS: ACTIVE]

---

## 1. Tổng quan Kiến trúc Luồng Đột biến Phiên Chat

Luồng 06 kết hợp triết lý **Offline-First** ở tầng Frontend với thiết kế **REST Idempotent + `O(1)` Cache Invalidation** ở tầng Backend cho hai động từ HTTP thay đổi trạng thái trên cùng một định danh tài nguyên `/api/v1/chat-sessions/:id`:

1. **Xóa mềm tương tác 2 bước (`DELETE /api/v1/chat-sessions/:id`)**:
   - Trên giao diện [`HistoryDrawer`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L24-L155), người dùng bấm nút thùng rác để mở hộp xác nhận nội tuyến `"Xóa?"` (`confirmDeleteId`) rồi bấm dấu tick xác nhận.
   - [`deleteSessionLocallyAndRemotely`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L144-L160) xóa ngay tin nhắn và phiên trong [`chatDb`](../../../frontend_rag/src/features/chat/db/chatDb.ts#L43) (IndexedDB) trước, cập nhật [`useSessionStore`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L148-L162) để chuyển ngay sang phiên gần nhất kế tiếp (hoặc tạo phiên trống mới), rồi mới gửi yêu cầu `DELETE` bất đồng bộ (*fire-and-forget*) khi đang online và đã đăng nhập.
   - Ở Backend, [`ChatSessionsService.remove`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L231-L240) luôn trả về `204 No Content` (kể cả khi ID không phải `UUIDv7`, không tồn tại, đã xóa trước đó, hoặc thuộc về người dùng khác). Khi có bản ghi thuộc sở hữu được cập nhật `deleted_at`, hệ thống tiếp tục gắn `deleted_at` cho toàn bộ tin nhắn thuộc phiên đó; MongoDB sẽ tự dọn vĩnh viễn sau **3 ngày (`259200` giây)** qua TTL index.
2. **Tự động đồng bộ tiêu đề nền (`PATCH /api/v1/chat-sessions/:id`)**:
   - Khi người dùng gửi câu hỏi đầu tiên ở chế độ local, [`createLocalSession`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L81-L102) cắt 48 ký tự đầu của câu hỏi làm tiêu đề cục bộ trong Dexie. Trong khi đó, Backend tạo phiên mới với tiêu đề timestamp mặc định (`YY-MM-DD:HH-MM-SS` UTC+7).
   - Ngay khi sự kiện SSE `done` trả về `session_id` từ Backend, [`useRag.send`](../../../frontend_rag/src/features/chat/store/ragStore.ts#L173-L175) gọi [`confirmBackendSession`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L174-L187) → [`reconcileBackendSession`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L105-L141) để ánh xạ ID local sang `backendId` và bắn yêu cầu nền [`renameRemoteSession`](../../../frontend_rag/src/features/chat/api/chatApi.ts#L83-L93) (`PATCH /api/v1/chat-sessions/:id`).
   - Backend kiểm tra độ dài tiêu đề theo Unicode codepoint (`1–120` ký tự), cập nhật tiêu đề nếu đúng chủ sở hữu và chưa xóa mềm, tăng bộ đếm thế hệ `userGen` (`O(1)`), và trả về `200 OK` kèm liên kết HATEOAS `_links`.

---

## 2. Sơ đồ Kiến trúc Tổng thể Chuẩn 26 Bước (Flowchart)

```mermaid
flowchart TB
  subgraph FE_IN["Giai đoạn 1 — Frontend Chiều đi (Client -> HTTP Request)"]
    CLI_IN["1. Client (Bấm Trash2 -> Check trên HistoryDrawer / SSE done -> confirmBackendSession)"]
    UI_IN["2. FE Component (HistoryDrawer 2-step confirm / ChatPanel)"]
    HK_IN["3. FE Hook & Store (useHistoryDrawerState -> useSessionStore.deleteSession / confirmBackendSession)"]
    SVC_IN["4. FE Service (chatSessionService: xoa/di tru Dexie chatDb truoc 0ms -> quyet dinh goi mang)"]
    DEXIE["IndexedDB chatDb: sessions & messages (Offline-First)"]
    API_IN["5. FE API (chatApi: deleteRemoteSession / renameRemoteSession)"]
    HTTP_IN["6. FE HTTP Client (http.ts: apiFetch DELETE / PATCH + Bearer)"]
    CLI_IN --> UI_IN --> HK_IN --> SVC_IN --> API_IN --> HTTP_IN
    SVC_IN <--> DEXIE
  end

  subgraph BE_IN["Giai đoạn 2 — Backend Chiều vào (HTTP Request -> Database)"]
    NODE["7. BE Node.js HTTP Server (main.ts: keepAliveTimeout=65s, headersTimeout=66s)"]
    MW["8. BE NestJS/Express Middleware (app.setup.ts: requestId, helmet, metrics, XML/JSON)"]
    GUARD["9. BE Guard (ThrottlerGuard 120/min + JwtAuthGuard token_version)"]
    INT_IN["10. BE Interceptor chiều vào (ContentNegotiation Accept 406 + LinksInterceptor metadata)"]
    PIPE["11. BE Pipe (ValidationPipe -> RenameSessionDto cho PATCH)"]
    CTRL_IN["12. BE Controller (ChatSessionsController: DELETE :id / PATCH :id)"]
    SVC_BE_IN["13. BE Service (ChatSessionsService: remove / rename, validTitle, invalidate O1 userGen)"]
    REPO_IN["14. BE Repository (ChatSessionsRepo.softDeleteOwned/renameOwnedLean + ChatMessagesRepo)"]
    MONGO["15. Database (MongoDB: chat_sessions & chat_messages TTL 3 ngay deleted_at)"]
    HTTP_IN -->|"DELETE / PATCH /api/v1/chat-sessions/:id"| NODE --> MW --> GUARD --> INT_IN --> PIPE --> CTRL_IN --> SVC_BE_IN --> REPO_IN --> MONGO
  end

  subgraph BE_OUT["Giai đoạn 3 — Backend Chiều về (Database -> HTTP Response)"]
    REPO_OUT["16. Database -> BE Repository (modifiedCount 0/1 hoac LeanChatSession)"]
    SVC_BE_OUT["17. BE Service (DELETE im lang void / PATCH 404 neu null hoac toSummary + invalidate)"]
    CTRL_OUT["18. BE Controller (tra 204 No Content hoac 200 SessionSummaryDto)"]
    INT_OUT["19. BE Interceptor chiều ra (LinksInterceptor +_links cho PATCH -> ContentNegotiation JSON/XML)"]
    EXP_OUT["20. BE NestJS/Express gửi response (HTTP 204 / 200 / HttpExceptionFilter + X-Request-Id)"]
    MONGO --> REPO_OUT --> SVC_BE_OUT --> CTRL_OUT --> INT_OUT --> EXP_OUT
  end

  subgraph FE_OUT["Giai đoạn 4 — Frontend Chiều về (HTTP Response -> Client)"]
    HTTP_OUT["21. FE HTTP Client (apiFetch kiem tra 401 auto-rotate -> tra Response)"]
    API_OUT["22. FE API (chatApi: res.ok void hoac parseJson RemoteSessionSummary / HttpError)"]
    SVC_OUT["23. FE Service (chatSessionService xac nhan dong bo nen khong rollback Dexie)"]
    HK_OUT["24. FE Hook & Store (useSessionStore cap nhat remaining[0] / backendId -> useHistoryDrawerState)"]
    UI_OUT["25. FE Component (HistoryDrawer go bo phien / ChatPanel cap nhat tieu de moi)"]
    CLI_OUT["26. Client (Giao dien phan hoi 0ms nhat quan ca online lan offline)"]
    EXP_OUT -->|"HTTP 204 No Content / 200 OK"| HTTP_OUT --> API_OUT --> SVC_OUT --> HK_OUT --> UI_OUT --> CLI_OUT
  end
```

---

## 3. Sơ đồ Tuần tự End-to-End (Sequence Diagram)

```mermaid
sequenceDiagram
  actor User as "Nguoi dung"
  participant HD as "HistoryDrawer / useRag"
  participant SS as "useSessionStore"
  participant CS as "chatSessionService"
  participant DB as "Dexie chatDb"
  participant API as "chatApi + apiFetch"
  participant BE as "NestJS Pipeline & Guard"
  participant CTRL as "ChatSessionsController"
  participant SVC as "ChatSessionsService"
  participant REPO as "Sessions & Messages Repo"
  participant MG as "MongoDB (TTL 3d)"

  rect rgb(235, 244, 255)
    Note over User,MG: "Nhanh A — Xoa mem tuong tac 2 buoc (DELETE /api/v1/chat-sessions/:id)"
    User->>HD: "Bam Trash2 -> mo confirmDeleteId ('Xoa?')"
    User->>HD: "Bam Check (Xac nhan xoa)"
    HD->>SS: "deleteSession(sessionId)"
    SS->>CS: "deleteSessionLocallyAndRemotely(sessionId, {isOffline, hasToken})"
    CS->>DB: "Xoa messages va session trong IndexedDB"
    CS-->>SS: "Tra ve danh sach remaining tu listLocalSessions()"
    SS->>SS: "Cap nhat sessions & chuyen active sang remaining[0] hoac startNewChat()"
    alt "Online va da dang nhap (!isOffline && hasToken && !USE_MOCK)"
      CS--)API: "Fire-and-forget: deleteRemoteSession(sessionId)"
      API->>BE: "DELETE /api/v1/chat-sessions/:id (Authorization: Bearer)"
      BE->>CTRL: "JwtAuthGuard -> remove(id, me)"
      CTRL->>SVC: "remove(me.id, id)"
      alt "Khong phai UUIDv7 (!isUuid(id))"
        SVC-->>CTRL: "Return som (void)"
      else "UUIDv7 hop le"
        SVC->>SVC: "invalidate(userId, id): bumpUserGen + delete detailCache"
        SVC->>REPO: "softDeleteOwned(userId, id, now)"
        REPO->>MG: "updateOne({_id, user_id, deleted_at: null}, {$set: {deleted_at: now}})"
        MG-->>REPO: "modifiedCount (1 hoac 0)"
        opt "modifiedCount > 0"
          SVC->>REPO: "softDeleteBySession(id, now)"
          REPO->>MG: "updateMany({session_id, deleted_at: null}, {$set: {deleted_at: now}})"
        end
        SVC-->>CTRL: "Return void (ke ca khi id la cua nguoi khac hoac da xoa)"
      end
      CTRL-->>API: "204 No Content (khong body)"
    else "Offline hoac chua dang nhap"
      CS-->>SS: "Bo qua goi mang, UI van giu trang thai da xoa"
    end
  end

  rect rgb(240, 250, 240)
    Note over User,MG: "Nhanh B — Tu dong dong bo tieu de nen (PATCH /api/v1/chat-sessions/:id)"
    HD->>SS: "SSE done(ev.session_id) -> confirmBackendSession(localId, backendId)"
    SS->>CS: "reconcileBackendSession(localId, backendId, {userId, hasToken})"
    CS->>DB: "Di tru localId -> backendId & dat synced=true"
    CS--)API: "Background: renameRemoteSession(backendId, session.title)"
    API->>BE: "PATCH /api/v1/chat-sessions/:id { title }"
    BE->>CTRL: "JwtAuthGuard + ValidationPipe -> rename(id, dto, me)"
    CTRL->>SVC: "rename(me.id, id, dto.title)"
    SVC->>SVC: "validTitle(title): trim + kiem tra 1..120 codepoints"
    alt "Tieu de khong hop le hoac thieu"
      SVC-->>API: "400 Bad Request ('invalid title')"
    else "Tieu de hop le"
      SVC->>REPO: "renameOwnedLean(userId, id, t)"
      REPO->>MG: "findOneAndUpdate({_id, user_id, deleted_at: null}, {$set: {title: t}})"
      alt "Khong tim thay / da xoa / khac chu so huu"
        MG-->>REPO: "null"
        SVC-->>API: "404 Not Found ('session not found')"
      else "Tim thay & da cap nhat"
        MG-->>REPO: "LeanChatSession moi"
        SVC->>SVC: "logEvent('SESSION_RENAMED') + invalidate(userId, id)"
        SVC-->>CTRL: "SessionSummaryDto (toSummary)"
        CTRL-->>API: "200 OK + HATEOAS _links (LinksInterceptor)"
      end
    end
  end
```

---

## 4. Phân rã Chi tiết 4 Giai đoạn Request–Response

### Giai đoạn 1 — Chiều đi Frontend: Client → FE → HTTP Request

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Client | [`HistoryDrawer.tsx`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L112-L147) (`<HistoryDrawer />`) | Click `Trash2`→`Check` / `done` | Xác nhận xóa 2 bước hoặc hoàn tất câu hỏi đầu kích hoạt đồng bộ tiêu đề. | UI click / SSE `done` |
| 2 | FE — Component | [`HistoryDrawer.tsx`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L112-L147) (`setConfirmDeleteId()`) | `session.id`, click `Check` | Click `Trash2` mở `"Xóa?"`, click `Check` gọi `deleteSession(session.id)`. | Gọi action từ Store |
| 3 | FE — Hook (+ Store) | [`sessionStore.ts`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L148-L187) (`useSessionStore.deleteSession()`) | `sessionId` / `(localId, backendId)` | Cập nhật `sessions` (chuyển sang `remaining[0]`) hoặc `confirmBackendSession()`. | Gọi `chatSessionService` |
| 4 | FE — Service | [`chatSessionService.ts`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L105-L160) (`deleteSessionLocallyAndRemotely()`) | `sessionId`, `isOffline`, `hasToken` | **DELETE**: xóa `chatDb` trước `0ms`; **PATCH**: di trú ID Dexie; online → gọi nền. | Dexie `0ms` + API nền |
| 5 | FE — API | [`chatApi.ts`](../../../frontend_rag/src/features/chat/api/chatApi.ts#L83-L100) (`deleteRemoteSession()`) | `sessionId`, `title` | Cấu hình `DELETE /chat-sessions/:id` hoặc `PATCH` `{ title }`, gọi `apiFetch()`. | `apiFetch(...)` |
| 6 | FE — HTTP Client (gọi fetch) | [`http.ts`](../../../frontend_rag/src/shared/api/http.ts#L23-L34) (`apiFetch()`) | Path, `accessToken` | Ghép `API_BASE` (`/api/v1`), gắn `Bearer` token và gửi `fetch()`. | HTTP `DELETE` / `PATCH` |

#### Ví dụ HTTP Request trên dây mạng

**Case A — Xóa mềm phiên chat (`DELETE`)**:
```http
DELETE /api/v1/chat-sessions/0193678a-4b12-7c80-9a11-2b3c4d5e6f70 HTTP/1.1
Host: localhost:3000
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Accept: application/json
```

**Case B — Cập nhật tiêu đề phiên chat (`PATCH`)**:
```http
PATCH /api/v1/chat-sessions/0193678a-4b12-7c80-9a11-2b3c4d5e6f70 HTTP/1.1
Host: localhost:3000
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Content-Type: application/json
Accept: application/json

{"title":"So sanh co che attention trong Transformer va Mamba"}
```

---

### Giai đoạn 2 — Chiều vào Backend: HTTP Request → BE → Database

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 7 | BE — Node.js HTTP Server | [`main.ts`](../../../backend_rag/src/main.ts#L1-L26) (`bootstrap()`) | Socket TCP `:3000` | Node `http.Server` (`keepAliveTimeout=65s`) nhận request vào Express. | Raw HTTP Request |
| 8 | BE — NestJS/Express Middleware | [`app.setup.ts`](../../../backend_rag/src/app.setup.ts#L40-L85) (`configureApp()`) | Raw Request | `trust proxy`, `X-Request-Id` UUIDv7, Helmet, metrics, parse JSON/XML (`415`), `/api/v1`. | `req.id`, `req.body` |
| 9 | BE — Guard | [`jwt-auth.guard.ts`](../../../backend_rag/src/modules/auth/guards/jwt-auth.guard.ts#L22-L47) (`JwtAuthGuard.canActivate()`) | `req.ip`, `Bearer` token | Rate limit `120 req/60s`; verify JWT + check `is_active` & `token_version` (`401`). | `req.user` (`AuthUser`) |
| 10 | BE — Interceptor chiều vào | [`xml.ts`](../../../backend_rag/src/common/http/xml.ts#L109-L119) (`ContentNegotiationInterceptor.intercept()`) | Header `Accept` | Kiểm tra `Accept` hợp lệ (`406` nếu sai); đọc `@WithLinks` cho `PATCH`. | Cho vào Pipe |
| 11 | BE — Pipe | [`chat-session.dto.ts`](../../../backend_rag/src/modules/chat-sessions/dto/chat-session.dto.ts#L69-L73) (`RenameSessionDto`) | `:id` & JSON body | `createValidationPipe()` kiểm tra `RenameSessionDto.title` cho `PATCH` (`400` nếu sai). | `id` & DTO sạch |
| 12 | BE — Controller | [`chat-sessions.controller.ts`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.controller.ts#L80-L108) (`ChatSessionsController.remove()`) | `id`, `me`, `dto` | `@Delete(':id')` gọi `remove()` (`204`); `@Patch(':id')` gọi `rename()`. | Gọi `ChatSessionsService` |
| 13 | BE — Service | [`chat-sessions.service.ts`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L101-L240) (`ChatSessionsService.remove()`) | `userId`, `id`, `title` | **DELETE**: `!isUuid` return; `invalidate()` (`O(1)` `bumpUserGen()`). **PATCH**: `validTitle()` `1..120` codepoints. | Lệnh Repository |
| 14 | BE — Repository | [`chat-sessions.repository.ts`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts#L20-L84) (`ChatSessionsRepository.softDeleteOwned()`), [`chat-messages.repository.ts`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-messages.repository.ts#L72-L77) (`ChatMessagesRepository.softDeleteBySession()`) | `ownedSessionFilter` | **DELETE**: `softDeleteOwned()` + `softDeleteBySession()` (`deleted_at`); **PATCH**: `renameOwnedLean()`. | Truy vấn Mongoose |
| 15 | Database | [`chat-session.schema.ts`](../../../backend_rag/src/modules/chat-sessions/schemas/chat-session.schema.ts#L5-L45) (`ChatSessionSchema`), [`chat-message.schema.ts`](../../../backend_rag/src/modules/chat-sessions/schemas/chat-message.schema.ts#L5-L54) (`ChatMessageSchema`) | Lệnh ghi qua Tunnel | Gắn `deleted_at` (kích hoạt TTL tự xóa sau `3 ngày` = `259200s`) hoặc cập nhật `title`. | Kết quả cập nhật |

---

### Giai đoạn 3 — Chiều về Backend: Database → BE → HTTP Response

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 16 | Database → BE — Repository | [`chat-sessions.repository.ts`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts#L71-L84) (`ChatSessionsRepository.renameOwnedLean()`) | Kết quả MongoDB | Trả `modifiedCount` (`0`/`1`) cho xóa mềm hoặc `LeanChatSession \| null` cho đổi tên. | Kết quả `.lean()` |
| 17 | BE — Service | [`chat-sessions.service.ts`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L222-L240) (`ChatSessionsService.rename()`) | `modifiedCount` / session | **DELETE**: im lặng `void` (idempotent). **PATCH**: `null` → `404` (không `403`); log + `invalidate()`. | `undefined` / `SessionSummaryDto` |
| 18 | BE — Controller | [`chat-sessions.controller.ts`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.controller.ts#L80-L108) (`ChatSessionsController.rename()`) | `void` / DTO | Trả `undefined` (`204 No Content`) hoặc `SessionSummaryDto` (`200 OK`). | `204` / DTO (`200`) |
| 19 | BE — Interceptor chiều ra | [`hateoas.ts`](../../../backend_rag/src/common/http/hateoas.ts#L40-L46) (`sessionLinks()`) | `undefined` / DTO | `204` giữ nguyên rỗng; `PATCH` gắn `_links: sessionLinks()` và serialize JSON/XML. | Rỗng / JSON kèm `_links` |
| 20 | BE — NestJS/Express gửi response | [`http-exception.filter.ts`](../../../backend_rag/src/common/http/http-exception.filter.ts#L69-L97) (`HttpExceptionFilter.catch()`) | Status `204`/`200` / lỗi | Gửi `204`/`200` (hoặc chuẩn hóa lỗi `400`/`401`/`404`) + `X-Request-Id` + metrics. | HTTP Response |

#### Ví dụ HTTP Response trên dây mạng

**Case A — Xóa mềm thành công hoặc lặp lại (`DELETE`)**:
```http
HTTP/1.1 204 No Content
X-Request-Id: req_0193678b9f0a71238c456789abcdef01
```

**Case B — Cập nhật tiêu đề thành công (`PATCH`)**:
```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
X-Request-Id: req_0193678b11227abc8def0123456789ab

{
  "id": "0193678a-4b12-7c80-9a11-2b3c4d5e6f70",
  "title": "So sanh co che attention trong Transformer va Mamba",
  "message_count": 2,
  "last_message_at": "2026-11-26T03:10:01.001Z",
  "created_at": "2026-11-26T03:10:00.000Z",
  "updated_at": "2026-11-26T03:10:02.150Z",
  "_links": {
    "self": { "href": "/api/v1/chat-sessions/0193678a-4b12-7c80-9a11-2b3c4d5e6f70" },
    "messages": { "href": "/api/v1/chat-sessions/0193678a-4b12-7c80-9a11-2b3c4d5e6f70/messages" },
    "rename": { "href": "/api/v1/chat-sessions/0193678a-4b12-7c80-9a11-2b3c4d5e6f70", "method": "PATCH" },
    "delete": { "href": "/api/v1/chat-sessions/0193678a-4b12-7c80-9a11-2b3c4d5e6f70", "method": "DELETE" },
    "chat": { "href": "/api/v1/rag/chat", "method": "POST" }
  }
}
```

---

### Giai đoạn 4 — Chiều về Frontend: HTTP Response → FE → Client

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 21 | FE — HTTP Client (kiểm tra status & đọc body) | [`http.ts`](../../../frontend_rag/src/shared/api/http.ts#L23-L34) (`apiFetch()`) | `Response` từ Backend | Auto-rotate `401` một lần và gửi lại request; chuyển `Response` (`204`/`200`) về `chatApi`. | `res: Response` |
| 22 | FE — API (xử lý dữ liệu) | [`chatApi.ts`](../../../frontend_rag/src/features/chat/api/chatApi.ts#L96-L100) (`deleteRemoteSession()`) | `res: Response` | `deleteRemoteSession()` check `res.ok` trả `void`; `renameRemoteSession()` đọc JSON DTO. | `void` / DTO |
| 23 | FE — Service | [`chatSessionService.ts`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L134-L138) (`reconcileBackendSession()`) | Kết quả đồng bộ | Gọi nền *fire-and-forget* xác nhận đồng bộ mà **không rollback** `Dexie` nếu rớt mạng. | Đồng bộ nền hoàn tất |
| 24 | FE — Hook (cập nhật state) | [`useChatWorkspace.ts`](../../../frontend_rag/src/features/chat/hooks/useChatWorkspace.ts#L61-L79) (`useHistoryDrawerState()`) | `remaining`, `backendId` | Store đã cập nhật `sessions` / `currentSessionId = backendId` từ trước, selector đẩy ra UI. | State Zustand nhất quán |
| 25 | FE — Component (hiển thị lại) | [`HistoryDrawer.tsx`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L83-L149) (`<HistoryDrawer />`) | `sessions`, `currentSessionId` | Gỡ phiên đã xóa (`0ms` không chờ DB) hoặc hiển thị tiêu đề mới trên `ChatPanel`. | DOM cập nhật tức thì |
| 26 | Client | [`HistoryDrawer.tsx`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L83-L149) (`<HistoryDrawer />`) | UI sau re-render | Thao tác xóa / đổi tên mượt mà tức thì kể cả khi mạng chậm hay offline. | Hoàn tất Offline-First |

---

## 5. Nhánh lỗi, Tính Idempotent & Các Bất biến Hệ thống

| Tình huống / Bất biến | Cơ chế thực thi & File tham chiếu | Hành vi chuẩn & Mã HTTP |
| :--- | :--- | :--- |
| **Xóa mềm Idempotent (`DELETE`)** | [`ChatSessionsService.remove`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L231-L240), [`ChatSessionsController.remove`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.controller.ts#L103-L108) | Gọi `DELETE` lần 1, lần 2+, với ID không phải UUID, ID không tồn tại, hoặc ID thuộc user khác đều **im lặng bỏ qua** và trả `204 No Content`. Không phát log `SESSION_DELETED` khi `modifiedCount === 0`. |
| **Ẩn quyền sở hữu khi Đổi tên (`PATCH`)** | [`ownedSessionFilter`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts#L20-L24), [`ChatSessionsService.rename`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L221-L223) | Truy vấn luôn gộp `{ _id: id, user_id: ObjectId(userId), deleted_at: null }`. Nếu ID sai định dạng UUID, đã xóa mềm, hoặc thuộc user khác, hệ thống trả `404 Not Found` (`session not found`), **không bao giờ trả `403 Forbidden`** để tránh lộ sự tồn tại tài nguyên. |
| **Kiểm tra tiêu đề Unicode (`1–120` ký tự)** | [`RenameSessionDto`](../../../backend_rag/src/modules/chat-sessions/dto/chat-session.dto.ts#L69-L73), [`validTitle`](../../../backend_rag/src/modules/chat-sessions/chat-session.mapper.ts#L11-L19) | `RenameSessionDto` bắt buộc kiểu chuỗi (`invalid title`). `validTitle` cắt khoảng trắng (`trim()`) và đếm độ dài bằng `Array.from(t).length` (mỗi emoji/ký tự Unicode là 1 ký tự). Chuỗi rỗng hoặc `> 120` ký tự ném `400 Bad Request` (`invalid title`). |
| **TTL 3 ngày tự động dọn dẹp** | [`SESSION_TTL_SECONDS`](../../../backend_rag/src/modules/chat-sessions/schemas/chat-session.schema.ts#L5), [`ChatSessionSchema.index`](../../../backend_rag/src/modules/chat-sessions/schemas/chat-session.schema.ts#L42-L45), [`ChatMessageSchema.index`](../../../backend_rag/src/modules/chat-sessions/schemas/chat-message.schema.ts#L51-L54) | Khi xóa mềm, cả `chat_sessions` và `chat_messages` được gắn cùng dấu thời gian `deleted_at: now`. MongoDB TTL background monitor tự xóa vĩnh viễn sau `3 * 24 * 60 * 60 = 259200` giây (3 ngày). |
| **Vô hiệu hóa Cache `O(1)` (`D-021`, `D-051`)** | [`bumpUserGen`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L101-L112), [`list` cache key](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L120-L130) | Các trang `listCache` được đặt khóa `${userId}:${gen}:...`. Khi đổi tên hoặc xóa phiên, `invalidate(userId, id)` chỉ cần tăng `userGen` của `userId` (`O(1)`) mà không phải duyệt `LRUCache` (10.000 mục, TTL `300_000` ms) để xóa từng trang lọc/sắp xếp, đồng thời xóa trực tiếp mục `${userId}:${id}` khỏi `sessionDetailCache`. |
| **Liên tục Offline-First ở Frontend (`D-017`)** | [`deleteSessionLocallyAndRemotely`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L144-L160), [`useSessionStore.deleteSession`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L148-L162) | Thao tác xóa trên UI luôn xóa Dexie trước và bỏ qua gọi `DELETE` từ xa khi `isOffline` là `true` hoặc chưa đăng nhập (`!hasToken`). Nếu Backend tạm mất kết nối trong lúc gọi nền, lỗi chỉ được ghi `console.warn` để không làm gián đoạn phiên làm việc hiện tại. |
