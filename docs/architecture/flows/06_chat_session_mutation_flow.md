# Luồng 06 — Quản lý Phiên Chat: Đổi tên & Xóa mềm Idempotent (`DELETE` & `PATCH /api/v1/chat-sessions/:id`)

- **Motivation/Background**: Người dùng cần xóa hoặc đồng bộ tiêu đề các phiên trò chuyện một cách tức thì trên giao diện ngay cả khi mất kết nối mạng, trong khi Backend phải bảo đảm tính toàn vẹn đa người dùng (không rò rỉ thông tin sở hữu), tính lũy đẳng (`Idempotent`) của phép xóa và vô hiệu hóa bộ nhớ đệm (`Cache Invalidation`) ở độ phức tạp `O(1)` cho danh sách phân trang nhiều chiều.
- **Purpose**: Tài liệu hóa chi tiết luồng Request–Response đầu-cuối cho hai thao tác ghi trên tài nguyên phiên chat: xóa mềm có xác nhận 2 bước (`DELETE /api/v1/chat-sessions/:id`) và tự động đồng bộ tiêu đề sau lượt hỏi đầu tiên (`PATCH /api/v1/chat-sessions/:id`).
- **Overview Pipeline**: Truy vết trực tiếp từ UI `HistoryDrawer` / `useRag` → Zustand Stores (`sessionStore`) → Dexie IndexedDB (`chatDb`) → `chatSessionService` → `chatApi` / `apiFetch` → NestJS Middleware & `JwtAuthGuard` → `ChatSessionsController` → `ChatSessionsService` (`invalidate` `O(1)` qua `userGen`) → `ChatSessionsRepository` & `ChatMessagesRepository` → MongoDB (TTL 3 ngày trên `deleted_at`) → Interceptors & HTTP Response.
- **Detailed Plan**: §1 Tổng quan kiến trúc luồng đột biến phiên chat; §2 Sơ đồ khối kiến trúc (Flowchart); §3 Sơ đồ tuần tự End-to-End (Sequence Diagram); §4 Phân rã 4 giai đoạn Request–Response & Wire Examples; §5 Các nhánh lỗi, ràng buộc Idempotency & Bất biến hệ thống.
- **References**: React 19, Zustand, Dexie.js (IndexedDB), NestJS 11, Mongoose/MongoDB (TTL Index), `lru-cache`, `class-validator`, `AGENTS.md`, `agents/rules/MD_CONVENTION.md`, `docs/api/API_DESIGN.md`.
- **Created**: 2026-11-26T10:12:00+07:00
- **Last Updated**: 2026-11-26T10:12:00+07:00

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

## 2. Sơ đồ Kiến trúc Tổng thể (Flowchart)

```mermaid
flowchart TB
  subgraph FE["Frontend (React 19 + Zustand + Dexie IndexedDB)"]
    UI["HistoryDrawer: 2-Step Inline Delete ('Xoa?' -> Check)"]
    SSE["useRag.send: SSE done -> confirmBackendSession"]
    STORE["useSessionStore: deleteSession / confirmBackendSession"]
    SVC_FE["chatSessionService: deleteSessionLocallyAndRemotely / reconcileBackendSession"]
    DEXIE["Dexie chatDb: sessions & messages (Offline-First)"]
    API_FE["chatApi + apiFetch: Bearer Token & 401 Single Rotate"]

    UI --> STORE
    SSE --> STORE
    STORE --> SVC_FE
    SVC_FE --> DEXIE
    SVC_FE --> API_FE
  end

  subgraph BE["Backend (NestJS 11 HTTP Pipeline)"]
    MW["Express Middleware: requestId, securityHeaders, metrics, xmlBody"]
    GUARD["JwtAuthGuard: Bearer Verify + is_active + token_version"]
    PIPE["ValidationPipe: whitelist + forbidNonWhitelisted + transform"]
    CTRL["ChatSessionsController: DELETE :id (204) & PATCH :id (200)"]
    INT["LinksInterceptor (_links) + ContentNegotiationInterceptor (JSON/XML)"]
    FLT["HttpExceptionFilter: ErrorBody + request_id"]

    MW --> GUARD --> PIPE --> CTRL --> INT
    GUARD -.-> FLT
    PIPE -.-> FLT
    CTRL -.-> FLT
  end

  subgraph CORE["Service, Cache O(1) & Persistence"]
    SVC_BE["ChatSessionsService: remove() / rename()"]
    CACHE["O(1) Cache Invalidation: bumpUserGen(userId) + sessionDetailCache.delete"]
    REPO_S["ChatSessionsRepository: softDeleteOwned / renameOwnedLean"]
    REPO_M["ChatMessagesRepository: softDeleteBySession"]
    MONGO["MongoDB: chat_sessions & chat_messages (TTL 3 ngay tren deleted_at)"]

    SVC_BE --> CACHE
    SVC_BE --> REPO_S
    SVC_BE --> REPO_M
    REPO_S --> MONGO
    REPO_M --> MONGO
  end

  API_FE -->|"DELETE / PATCH /api/v1/chat-sessions/:id"| MW
  CTRL --> SVC_BE
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
| 1.1 | UI Component (`DELETE`) | [`HistoryDrawer`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L112-L147) | Click icon `Trash2` rồi click icon `Check` | Cơ chế xác nhận 2 bước tại chỗ (`confirmDeleteId`): click lần 1 mở badge `"Xóa?"` (`e.stopPropagation()` ngăn kích hoạt `switchSession`); click `Check` đặt `confirmDeleteId(null)` và gọi `void deleteSession(session.id)`. | Gọi `deleteSession(session.id)` bất đồng bộ. |
| 1.2 | SSE Trigger (`PATCH`) | [`useRag.send`](../../../frontend_rag/src/features/chat/store/ragStore.ts#L171-L175) | Sự kiện SSE `done` chứa `ev.session_id` | Khi stream trả lời hoàn tất và Backend cấp `ev.session_id`, `ragStore` gọi `await useSessionStore.getState().confirmBackendSession(sessionId, ev.session_id)`. | Kích hoạt đối soát ID và đồng bộ tiêu đề. |
| 1.3 | Hook & Zustand Store | [`useHistoryDrawerState`](../../../frontend_rag/src/features/chat/hooks/useChatWorkspace.ts#L61-L79), [`useSessionStore.deleteSession`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L148-L162), [`useSessionStore.confirmBackendSession`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L174-L187) | `sessionId` (hoặc `localId`, `backendId`) | `deleteSession` gọi `deleteSessionLocallyAndRemotely`, cập nhật `sessions: remaining`; nếu xóa đúng `currentSessionId` thì chuyển ngay sang `remaining[0].id` hoặc gọi `startNewChat()`. `confirmBackendSession` gọi `reconcileBackendSession` và cập nhật `currentSessionId` sang `backendId` nếu vừa di trú từ ID cục bộ. | State Zustand cập nhật tức thì trên UI. |
| 1.4 | Offline-First Service & IndexedDB | [`deleteSessionLocallyAndRemotely`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L144-L160), [`reconcileBackendSession`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L105-L141), [`chatDb`](../../../frontend_rag/src/features/chat/db/chatDb.ts#L30-L46) | `sessionId`, `{ isOffline, hasToken }` / `{ userId, hasToken }` | **Luồng DELETE**: Xóa sạch `chatDb.messages` theo `sessionId` và xóa `chatDb.sessions` trước; trả `remaining` qua `listLocalSessions()`. Nếu `hasToken && !USE_MOCK && !opts.isOffline`, gọi nền `void deleteRemoteSession(sessionId)`. **Luồng PATCH**: Di trú bản ghi Dexie từ `localId` sang `backendId` (đặt `synced: true`); nếu `hasToken && !USE_MOCK && session.title`, gọi nền `void renameRemoteSession(backendId, session.title)`. | Dexie nhất quán; gọi HTTP ở nền (*fire-and-forget* có `.catch(console.warn)`). |
| 1.5 | Feature API & HTTP Client | [`deleteRemoteSession`](../../../frontend_rag/src/features/chat/api/chatApi.ts#L96-L100), [`renameRemoteSession`](../../../frontend_rag/src/features/chat/api/chatApi.ts#L83-L93), [`apiFetch`](../../../frontend_rag/src/shared/api/http.ts#L23-L34) | `sessionId`, `title` (cho `PATCH`) | `deleteRemoteSession` gửi `DELETE /chat-sessions/:id` không body; `renameRemoteSession` gửi `PATCH /chat-sessions/:id` với `Content-Type: application/json` và `JSON.stringify({ title })`. `apiFetch` gắn `Authorization: Bearer <accessToken>`, tự động `rotateTokens` một lần nếu gặp `401` rồi gửi lại request. | HTTP Request gửi tới `${API_BASE}/chat-sessions/:id`. |

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

### Giai đoạn 2 — Chiều vào Backend: HTTP Request → BE → Repository → Database

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 2.1 | Express Middleware Stack | [`configureApp`](../../../backend_rag/src/app.setup.ts#L40-L85), [`unsupportedBody` / `xmlBodyMiddleware`](../../../backend_rag/src/common/http/xml.ts#L70-L105) | Raw HTTP Request | Thiết lập `trust proxy`, gắn `requestIdMiddleware`, `securityHeaders`, ghi nhận `PrometheusMetricsRegistry`, kiểm tra `unsupportedBody` (`415` nếu có body khác JSON/XML), giải mã XML (`textBody` max `100kb` → `xmlBodyMiddleware` chống XXE), áp dụng global prefix `/api/v1`. | `req` đã có `request_id` và `req.body` chuẩn hóa. |
| 2.2 | Authentication Guard | [`JwtAuthGuard.canActivate`](../../../backend_rag/src/modules/auth/guards/jwt-auth.guard.ts#L22-L47), [`CurrentUser`](../../../backend_rag/src/modules/auth/decorators/current-user.decorator.ts#L6-L9) | Header `Authorization: Bearer <token>` | `bearerToken(req)` tách JWT; `tokens.verifyAccess(token)` xác thực chữ ký & hạn; `auth.getActiveUser(claims.id)` kiểm tra `user.is_active` và khớp `user.token_version === claims.tokenVersion`. Nếu lỗi ném `401 Unauthorized`. | Gắn `req.user = { id, roles, email }`. |
| 2.3 | Global Validation Pipe | [`createValidationPipe`](../../../backend_rag/src/common/http/validation.ts#L18-L26), [`RenameSessionDto`](../../../backend_rag/src/modules/chat-sessions/dto/chat-session.dto.ts#L69-L73) | `:id` param & JSON/XML body | `ValidationPipe` (`whitelist: true`, `forbidNonWhitelisted: true`, `transform: true`). Với `DELETE`, không có DTO body. Với `PATCH`, xác thực `RenameSessionDto.title` là chuỗi (`@IsString({ message: 'invalid title' })`), loại bỏ field thừa; lỗi ném `400 Bad Request`. | Tham số `id` và `dto` sạch. |
| 2.4 | Controller | [`ChatSessionsController.remove`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.controller.ts#L103-L108), [`ChatSessionsController.rename`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.controller.ts#L80-L90) | `id: string`, `me: AuthUser`, `dto?: RenameSessionDto` | `@UseGuards(JwtAuthGuard)` bảo vệ cả class. `remove` được gắn `@Delete(':id')` + `@HttpCode(HttpStatus.NO_CONTENT)` gọi `this.sessions.remove(me.id, id)`. `rename` được gắn `@Patch(':id')` + `@WithLinks(...)` gọi `this.sessions.rename(me.id, id, dto.title)`. | Ủy quyền xuống `ChatSessionsService`. |
| 2.5 | Service & `O(1)` Cache Invalidation | [`ChatSessionsService.remove`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L231-L240), [`ChatSessionsService.rename`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L215-L228), [`ChatSessionsService.invalidate`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L101-L112) | `userId`, `id`, `title` (cho `PATCH`) | **DELETE**: Nếu `!isUuid(id)` return ngay (vẫn `204`). Gọi `invalidate(userId, id)` (tăng `userGen` của `userId` thêm `1` và xóa `sessionDetailCache` theo key `${userId}:${id}`) trước khi gọi repo. **PATCH**: `validTitle(title)` cắt khoảng trắng đầu/cuối (`trim()`) và kiểm tra `1..120` Unicode codepoints (`Array.from(t).length`), kiểm tra `isUuid(id)`, gọi repo đổi tên; nếu cập nhật thành công mới ghi log `SESSION_RENAMED` và gọi `invalidate(userId, id)`. | Thực thi lệnh ghi xuống MongoDB và làm mới cache `O(1)`. |
| 2.6 | Repository & MongoDB | [`ChatSessionsRepository`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-sessions.repository.ts#L20-L84), [`ChatMessagesRepository`](../../../backend_rag/src/modules/chat-sessions/repositories/chat-messages.repository.ts#L72-L77), [`ChatSessionSchema`](../../../backend_rag/src/modules/chat-sessions/schemas/chat-session.schema.ts#L36-L45), [`ChatMessageSchema`](../../../backend_rag/src/modules/chat-sessions/schemas/chat-message.schema.ts#L45-L54) | `ownedSessionFilter(userId, id)` (`{ _id: id, user_id: ObjectId(userId), deleted_at: null }`) | **DELETE**: `softDeleteOwned` chạy `updateOne` đặt `deleted_at: now`. Nếu `modifiedCount > 0`, gọi tiếp `messages.softDeleteBySession(id, now)` (`updateMany` trên `chat_messages` có `deleted_at: null`) và ghi event `SESSION_DELETED`. Cả hai collection đều có TTL index `{ deleted_at: 1 }` với `expireAfterSeconds: 259200` (3 ngày). **PATCH**: `renameOwnedLean` chạy `findOneAndUpdate(ownedSessionFilter, { $set: { title } }, { new: true }).lean()`. | `modifiedCount` (`0`/`1`) hoặc `LeanChatSession \| null`. |

---

### Giai đoạn 3 — Chiều về Backend: Database → Repository → Service → Controller → Interceptor → HTTP Response

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 3.1 | Repository → Service | [`ChatSessionsService.remove`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L235-L240), [`ChatSessionsService.rename`](../../../backend_rag/src/modules/chat-sessions/chat-sessions.service.ts#L222-L228), [`toSummary`](../../../backend_rag/src/modules/chat-sessions/chat-session.mapper.ts#L33-L45) | `modifiedCount` (`DELETE`) hoặc `LeanChatSession \| null` (`PATCH`) | **DELETE**: Không ném lỗi dù `modifiedCount === 0` (đã xóa hoặc không thuộc quyền sở hữu), kết thúc trả `Promise<void>`. **PATCH**: Nếu `s === null`, ném `NotFoundException('session not found')` (`404`); nếu có bản ghi, chuyển đổi qua `toSummary(s)` thành `SessionSummaryDto` (chuẩn hóa timestamp ISO 8601). | `undefined` (`DELETE`) hoặc `SessionSummaryDto` (`PATCH`). |
| 3.2 | HATEOAS Interceptor | [`LinksInterceptor`](../../../backend_rag/src/common/http/hateoas.ts#L157-L177), [`sessionLinks`](../../../backend_rag/src/common/http/hateoas.ts#L40-L46) | Metadata `@WithLinks` & response body | **DELETE**: Handler không có `@WithLinks` và body là `undefined` → đi qua không thay đổi. **PATCH**: Đọc metadata `@WithLinks({ resource: (s) => sessionLinks(s.id) })`, gọi `addLinks` tạo bản sao object và gắn `_links` (`self`, `messages`, `rename`, `delete`, `chat`). | Body không đổi (`undefined`) hoặc có `_links`. |
| 3.3 | Content Negotiation Interceptor | [`ContentNegotiationInterceptor`](../../../backend_rag/src/common/http/xml.ts#L109-L127) | Header `Accept` & body từ `LinksInterceptor` | Kiểm tra `req.accepts(OFFERED)` (nếu không khớp trả `406 Not Acceptable`). Nếu `body === undefined` (`204 No Content` của `DELETE`) hoặc client không ưu tiên XML → giữ nguyên. Nếu client yêu cầu `application/xml` trên `PATCH` → chuyển sang XML bằng `toXml(body)` với root `<response>`. | JSON / XML hoặc rỗng (`204`). |
| 3.4 | Exception Filter (nếu có lỗi) | [`HttpExceptionFilter`](../../../backend_rag/src/common/http/http-exception.filter.ts#L69-L97) | `HttpException` hoặc lỗi hệ thống | Chuyển mọi ngoại lệ (`400`, `401`, `404`, `405`, `406`, `415`, `500`) về chuẩn `ErrorBody` (`{ statusCode, error, message, request_id }`) và serialize JSON/XML theo `Accept`. | HTTP Error Response đồng nhất. |

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
| 4.1 | HTTP Client | [`apiFetch`](../../../frontend_rag/src/shared/api/http.ts#L23-L34) | `Response` từ Backend | Nếu nhận `401` khi đã có token, thực thi `rotateTokens(token)` một lần và gửi lại request với access token mới. Trả `Response` về `chatApi`. | `Response` (`204`, `200`, hoặc lỗi). |
| 4.2 | Feature API | [`deleteRemoteSession`](../../../frontend_rag/src/features/chat/api/chatApi.ts#L96-L100), [`renameRemoteSession`](../../../frontend_rag/src/features/chat/api/chatApi.ts#L83-L93) | `Response` | `deleteRemoteSession` kiểm tra `res.ok` (`204` thỏa mãn `res.ok`), trả `Promise<void>`; nếu `!res.ok` ném `HttpError`. `renameRemoteSession` gọi `parseJson<RemoteSessionSummary>` đọc JSON trả về; nếu `!res.ok` ném `HttpError`. | `void` hoặc `RemoteSessionSummary`. |
| 4.3 | Background Reconciliation | [`deleteSessionLocallyAndRemotely`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L153-L157), [`reconcileBackendSession`](../../../frontend_rag/src/features/chat/services/chatSessionService.ts#L134-L138) | `Promise<void>` / `Promise<RemoteSessionSummary>` | Do cả hai lệnh gọi mạng đều chạy ở nền (*fire-and-forget* gắn `.catch((err) => console.warn(...))`), giao diện không bị chặn bởi độ trễ mạng hay đường hầm MongoDB và không rollback dữ liệu IndexedDB khi offline. | Hoàn tất đồng bộ nền không block UI. |
| 4.4 | Zustand & React UI | [`useSessionStore`](../../../frontend_rag/src/features/chat/store/sessionStore.ts#L153-L186), [`HistoryDrawer`](../../../frontend_rag/src/features/chat/components/HistoryDrawer.tsx#L83-L149) | Danh sách `sessions` & `currentSessionId` đã cập nhật | Trong luồng xóa, người dùng đã thấy phiên biến mất ngay khỏi danh sách `HistoryDrawer` và khung chat chuyển sang phiên kế tiếp (`switchSession(remaining[0].id)`) hoặc màn hình trắng mới (`startNewChat()`) ngay từ Bước 1.3–1.4. | Trải nghiệm phản hồi 0ms, nhất quán offline-first. |

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
