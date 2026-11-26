# Luồng 02 — Đăng nhập, Chống Brute-Force & Cấp phát Cặp Token (POST /api/v1/auth/tokens)

- **Motivation/Background**: Trong nền tảng `RAG_TS`, chức năng đăng nhập là cổng xác thực chính để chuyển đổi thông tin định danh (`email`, `password`) thành cặp Access Token (`JWT`) và Refresh Token lưu trữ trạng thái phiên trên trình duyệt; do MongoDB từ xa đi qua Cloudflare Access TCP tunnel (`~15–25ms` mỗi RTT) và mật khẩu dùng `Argon2id` tiêu tốn CPU, luồng này phải kết hợp song song hóa truy vấn cache-miss, bộ nhớ đệm `LRUCache`, chống brute-force hai tầng và cơ chế `dummyHash` chống rò rỉ thông tin qua thời gian phản hồi.
- **Purpose**: Đặc tả chi tiết luồng request-response đầu-cuối (end-to-end) của thao tác đăng nhập tại `POST /api/v1/auth/tokens`, ánh xạ từng bước xử lý qua toàn bộ các tầng Frontend và Backend cùng các bất biến bảo mật và kịch bản lỗi.
- **Overview Pipeline**: Client thao tác biểu mẫu trong `AuthOverlay` → `useAuthSession` / `useAuth` (`Zustand`) → `authService` → `authApi` / `publicFetch` → `HTTP POST /api/v1/auth/tokens` → Express Middleware (`trust proxy`, `requestId`, `helmet`, `metrics`, `XML/JSON`) → `ThrottlerGuard` (`@RateLimit('login')`) → `ValidationPipe` → `AuthController.login` → `AuthService.login` (`LRUCache` + `Promise.all` MongoDB + `Argon2id`/`dummyHash` + `TokenService.issue`) → `LinksInterceptor` & `ContentNegotiationInterceptor` → HTTP `201 Created` → `AuthApi` / `authStore` cập nhật `localStorage` và chuyển giao diện `ChatPanel`.
- **Detailed Plan**: §1 Kiến trúc Tổng thể qua các Tầng; §2 Biểu đồ Tuần tự End-to-End (Sequence Diagram); §3 Chi tiết 4 Giai đoạn Request–Response & Wire Examples; §4 Xử lý Lỗi & Các Bất biến Hệ thống (Error Paths & System Invariants).
- **References**: [AGENTS.md](../../../AGENTS.md), [MD_CONVENTION.md](../../../agents/rules/MD_CONVENTION.md), [AUTH_AND_SECURITY_ARCHITECTURE.md](../../../backend_rag/docs/AUTH_AND_SECURITY_ARCHITECTURE.md), [API_DESIGN.md](../../api/API_DESIGN.md), [AuthOverlay.tsx](../../../frontend_rag/src/features/auth/AuthOverlay.tsx), [auth.controller.ts](../../../backend_rag/src/modules/auth/auth.controller.ts), [auth.service.ts](../../../backend_rag/src/modules/auth/auth.service.ts), [token.service.ts](../../../backend_rag/src/modules/auth/token.service.ts).
- **Created**: 2026-11-26T10:12:00+07:00
- **Last Updated**: 2026-11-26T12:00:00+07:00

[STATUS: ACTIVE]

---

## 1. Kiến trúc Tổng thể qua các Tầng

Sơ đồ dưới đây ánh xạ đầy đủ **26 bước** trong pipeline chuẩn (`Client → FE [Component → Hook → Service → API → HTTP Client] → HTTP Request → BE [Node.js HTTP Server → Middleware → Guard → Interceptor In → Pipe → Controller → Service → Repository → Database] → BE Return [Repository → Service → Controller → Interceptor Out → NestJS/Express gửi response] → HTTP Response → FE Return [HTTP Client → API → Service → Hook → Component] → Client`) vào đúng các file mã nguồn thực tế cho endpoint `POST /api/v1/auth/tokens`.

```mermaid
flowchart TB
  subgraph FE_IN["Giai đoạn 1 — Frontend Chiều đi (Client -> HTTP Request)"]
    CLI_IN["1. Client (Người dùng nhập email, password)"]
    CMP_IN["2. FE Component (ChatPanel -> AuthOverlay.handleSubmit)"]
    HK_IN["3. FE Hook & Store (useAuthSession -> authStore.login)"]
    FSVC_IN["4. FE Service (authService.ts: authenticate)"]
    FAPI_IN["5. FE API (authApi.ts: loginWithPassword)"]
    HTTP_IN["6. FE HTTP Client (http.ts: publicFetch -> POST /api/v1/auth/tokens)"]
    CLI_IN --> CMP_IN --> HK_IN --> FSVC_IN --> FAPI_IN --> HTTP_IN
  end

  subgraph BE_IN["Giai đoạn 2 — Backend Chiều vào (HTTP Request -> Database)"]
    NODE["7. BE Node.js HTTP Server (main.ts: UV_THREADPOOL_SIZE=4, keepAliveTimeout=65s)"]
    MW["8. BE NestJS/Express Middleware (app.setup.ts: trust proxy, requestId, helmet, metrics, XML/JSON)"]
    GD["9. BE Guard (ThrottlerGuard + @RateLimit login 10 req/60s)"]
    INT_IN["10. BE Interceptor chiều vào (ContentNegotiation Accept 406 + LinksInterceptor metadata)"]
    PP["11. BE Pipe (ValidationPipe whitelist/transform -> LoginDto)"]
    CTL_IN["12. BE Controller (AuthController.login @Post tokens)"]
    BESVC_IN["13. BE Service (AuthService.login LRUCache, lockout, Argon2id/dummyHash & TokenService.issue)"]
    REPO_IN["14. BE Repository (LoginAttemptsRepository, UsersRepository, RefreshTokensRepository)"]
    DB["15. Database (MongoDB: login_attempts, users, refresh_tokens)"]
    HTTP_IN -->|"POST /api/v1/auth/tokens"| NODE --> MW --> GD --> INT_IN --> PP --> CTL_IN --> BESVC_IN --> REPO_IN --> DB
  end

  subgraph BE_OUT["Giai đoạn 3 — Backend Chiều về (Database -> HTTP Response)"]
    REPO_OUT["16. Database -> BE Repository (LeanUser, attemptCount, insert RefreshToken)"]
    BESVC_OUT["17. BE Service (cập nhật cache, lọc password_hash -> AuthResponseDto)"]
    CTL_OUT["18. BE Controller (AuthController.login trả AuthResponseDto status 201)"]
    INT_OUT["19. BE Interceptor chiều ra (LinksInterceptor +_links -> ContentNegotiation JSON/XML)"]
    EXP_OUT["20. BE NestJS/Express gửi response (HTTP 201 / HttpExceptionFilter + X-Request-Id + Metrics)"]
    DB --> REPO_OUT --> BESVC_OUT --> CTL_OUT --> INT_OUT --> EXP_OUT
  end

  subgraph FE_OUT["Giai đoạn 4 — Frontend Chiều về (HTTP Response -> Client)"]
    HTTP_OUT["21. FE HTTP Client (publicFetch nhận Response HTTP 201 / 401 / 429)"]
    FAPI_OUT["22. FE API (authApi.ts: readJsonOrThrow -> AuthTokenResponse hoặc AuthApiError)"]
    FSVC_OUT["23. FE Service (authService.ts: trả AuthTokenResponse hoặc mapAuthError tiếng Việt)"]
    HK_OUT["24. FE Hook & Store (authStore set + persist localStorage -> useAuthSession)"]
    CMP_OUT["25. FE Component (AuthOverlay tắt loading -> ChatPanel + MessageList + initSessions)"]
    CLI_OUT["26. Client (Người dùng đăng nhập thành công / thấy thông báo lỗi tiếng Việt)"]
    EXP_OUT -->|"HTTP 201 Created / 4xx"| HTTP_OUT --> FAPI_OUT --> FSVC_OUT --> HK_OUT --> CMP_OUT --> CLI_OUT
  end
```

---

## 2. Biểu đồ Tuần tự End-to-End (Sequence Diagram)

Biểu đồ tuần tự mô tả toàn bộ vòng đời một lần đăng nhập, bao gồm bước đọc song song `Promise.all` khi cache-miss, cơ chế xác thực `Argon2id` kết hợp `dummyHash` chống timing attack, ghi nhận lần thử thất bại hoặc cấp phát cặp token mới và cập nhật trạng thái phiên ở Frontend.

```mermaid
sequenceDiagram
  autonumber
  actor User as "Client / Browser"
  participant AuthUI as "AuthOverlay & ChatPanel"
  participant Store as "useAuth (Zustand)"
  participant AuthFE as "authService & authApi"
  participant HTTP as "publicFetch"
  participant Edge as "Express MW / Throttler / ValidationPipe"
  participant Ctrl as "AuthController.login"
  participant Svc as "AuthService.login"
  participant Cache as "LRUCache (AuthService)"
  participant Tok as "TokenService.issue"
  participant DB as "MongoDB (Repositories)"

  User->>AuthUI: "Nhập email, password và submit form"
  AuthUI->>Store: "login(email, password)"
  Store->>AuthFE: "authenticate(email, password)"
  AuthFE->>HTTP: "publicFetch('/auth/tokens', POST JSON)"
  HTTP->>Edge: "POST /api/v1/auth/tokens"

  alt "Vượt ngưỡng IP Rate Limit (10/min)"
    Edge-->>HTTP: "429 Too Many Requests (Retry-After)"
  else "Body sai định dạng hoặc Content-Type không hợp lệ"
    Edge-->>HTTP: "400 Bad Request / 415 Unsupported Media Type"
  else "Hợp lệ qua Middleware, Guard & Pipe"
    Edge->>Ctrl: "login(LoginDto, ip)"
    Ctrl->>Svc: "login(dto, ip)"
    Svc->>Svc: "normalizeEmail & sha256(email|ip)"
    Svc->>Cache: "Đọc attemptCountCache.get(key) & userByEmailCache.get(email)"

    alt "Cache miss một hoặc cả hai giá trị"
      par "Truy vấn MongoDB song song (Promise.all)"
        Svc->>DB: "LoginAttemptsRepository.countByKey(key)"
        Svc->>DB: "UsersRepository.findByEmailLean(email)"
      end
      DB-->>Svc: "dbAttempts, dbUser"
      Svc->>Cache: "Nạp attemptCountCache & userByEmailCache"
    end

    alt "attemptCount >= MAX_LOGIN_FAILURES (5)"
      Svc-->>Edge: "HttpException 429 ('too many login attempts')"
      Edge-->>HTTP: "429 Too Many Requests (HttpExceptionFilter)"
    else "Chưa chạm ngưỡng khóa tài khoản"
      Svc->>Svc: "argon2.verify(user ? user.password_hash : dummyHash(), dto.password)"

      alt "!user || !ok (Sai email hoặc sai mật khẩu)"
        Svc->>Cache: "attemptCountCache.set(key, attemptCount + 1)"
        Svc->>DB: "LoginAttemptsRepository.record(key)"
        Svc-->>Edge: "UnauthorizedException 401 ('invalid credentials')"
        Edge-->>HTTP: "401 Unauthorized (HttpExceptionFilter)"
      else "!user.is_active (Tài khoản bị khóa bởi Admin)"
        Svc-->>Edge: "UnauthorizedException 401 ('invalid credentials')"
        Edge-->>HTTP: "401 Unauthorized (HttpExceptionFilter)"
      else "Xác thực thành công & is_active = true"
        Svc->>Svc: "Reset attemptCountCache nếu hadFailures"
        Svc->>DB: "UsersRepository.updateLastLogin(user._id, now)"
        par "Cấp phát token & dọn lịch sử lỗi song song"
          Svc->>Tok: "issue(user, familyId)"
          Tok->>Tok: "jwt.signAsync(sub, roles, ver)"
          Tok->>DB: "RefreshTokensRepository.create(token_hash, family_id, expires_at)"
          Svc->>DB: "LoginAttemptsRepository.clearByKey(key) (nếu hadFailures)"
        end
        Svc-->>Ctrl: "AuthResponseDto (access_token, refresh_token, user)"
        Ctrl-->>Edge: "LinksInterceptor (+_links) & ContentNegotiationInterceptor"
        Edge-->>HTTP: "201 Created (JSON hoặc XML + X-Request-Id)"
      end
    end
  end

  alt "Phản hồi lỗi 401 / 429 / 4xx"
    HTTP-->>AuthFE: "Throw AuthApiError -> mapAuthError()"
    AuthFE-->>Store: "Throw Error(Thông báo tiếng Việt)"
    Store-->>AuthUI: "setError(message) hiển thị AlertTriangle"
  else "Phản hồi 201 Created"
    HTTP-->>AuthFE: "AuthTokenResponse"
    AuthFE-->>Store: "set({ user, accessToken, refreshToken, isAuthenticated: true })"
    Store->>Store: "Persist vào localStorage ('rag-auth-storage')"
    Store-->>AuthUI: "ChatPanel chuyển từ AuthOverlay sang MessageList & gọi initSessions()"
  end
```

---

## 3. Chi tiết 4 Giai đoạn Request–Response & Wire Examples

### 3.1. Giai đoạn 1 — Chiều đi Frontend: Client → FE → HTTP Request

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | Client | [`ChatPanel.tsx`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L110-L118) | Người dùng chưa đăng nhập (`isAuthenticated = false`), nhập `email`, `password` | Client mở giao diện chat, thấy `<AuthOverlay />` (`data-testid="chat-auth-panel"`), điền thông tin đăng nhập và nhấn nút `Đăng nhập`. | Sự kiện submit biểu mẫu trên trình duyệt |
| 2 | FE — Component | [`AuthOverlay.tsx`](../../../frontend_rag/src/features/auth/AuthOverlay.tsx#L21-L36) (`AuthOverlay.handleSubmit`) | Sự kiện submit form (`email`, `password`, `isLogin = true`) | Gọi `e.preventDefault()`, xóa lỗi cũ (`setError('')`), bật trạng thái `loading = true`, kiểm tra HTML constraint (`required`, `type="email"`, `minLength={8}` ở [`AuthOverlay.tsx`](../../../frontend_rag/src/features/auth/AuthOverlay.tsx#L107-L139)) và gọi `await login(email, password)`. | Lời gọi bất đồng bộ tới Hook `login(email, password)` |
| 3 | FE — Hook (+ Store) | [`useAuth.ts`](../../../frontend_rag/src/features/auth/hooks/useAuth.ts#L4-L22) (`useAuthSession`) & [`authStore.ts`](../../../frontend_rag/src/features/auth/authStore.ts#L40-L48) (`useAuth.login`) | Selector từ `useAuth` | `useAuthSession` trích xuất hàm `login` cho UI mà không subscribe dư thừa các field khác; `authStore.login` ủy quyền xử lý đăng nhập cho `authenticate(email, password)` tại tầng service. | Lời gọi `authenticate(email, password)` |
| 4 | FE — Service | [`authService.ts`](../../../frontend_rag/src/features/auth/services/authService.ts#L36-L42) (`authenticate`) | `email`, `password` | Bọc lời gọi `loginWithPassword({ email, password })` trong `try/catch` để ánh xạ mọi lỗi xác thực/mạng qua `mapAuthError` sang thông điệp tiếng Việt thân thiện. | Lời gọi `loginWithPassword` kèm xử lý lỗi |
| 5 | FE — API | [`authApi.ts`](../../../frontend_rag/src/features/auth/api/authApi.ts#L46-L53) (`loginWithPassword`) | `payload: LoginPayload` (`{ email, password }`) | Gọi `publicFetch('/auth/tokens', ...)` với `method: 'POST'`, header `Content-Type: application/json` và `JSON.stringify(payload)` (cố ý không dùng `apiFetch` để không gắn Bearer token cũ). | Yêu cầu tới `publicFetch('/auth/tokens', init)` |
| 6 | FE — HTTP Client (gọi fetch) | [`http.ts`](../../../frontend_rag/src/shared/api/http.ts#L15-L17) (`publicFetch`) & [`config.ts`](../../../frontend_rag/src/shared/api/config.ts#L1) (`API_BASE`) | `path = '/auth/tokens'`, `RequestInit` | Ghép `API_BASE` (`/api/v1`) với `/auth/tokens` và gọi `fetch('/api/v1/auth/tokens', init)`. | HTTP Request `POST /api/v1/auth/tokens` rời trình duyệt |

#### Wire Example — HTTP Request (`POST /api/v1/auth/tokens`)

```http
POST /api/v1/auth/tokens HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Accept: application/json

{
  "email": "researcher@university.edu",
  "password": "correct horse battery"
}
```

---

### 3.2. Giai đoạn 2 — Chiều vào Backend: HTTP Request → BE → Database

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 7 | BE — Node.js HTTP Server | [`main.ts`](../../../backend_rag/src/main.ts#L1-L24) (`bootstrap`) | `process.env`, HTTP socket cổng `3000` | Đặt `UV_THREADPOOL_SIZE = '4'` ở dòng 1 cho các tác vụ `Argon2id` trên libuv threadpool; validate env qua `parseEnv`; mở Node HTTP Server (`keepAliveTimeout = 65_000`, `headersTimeout = 66_000`). | Kết nối HTTP đi vào ứng dụng NestJS/Express |
| 8 | BE — NestJS/Express Middleware | [`app.setup.ts`](../../../backend_rag/src/app.setup.ts#L40-L75) & [`xml.ts`](../../../backend_rag/src/common/http/xml.ts#L70-L105) | Express `Request` | Cấu hình `trust proxy` lấy đúng IP client cho rate limiter & brute-force key; chạy `requestIdMiddleware` (`X-Request-Id` UUIDv7 + `AsyncLocalStorage`), `securityHeaders` (Helmet), `PrometheusMetricsRegistry.middleware`, `unsupportedBody` (`415`), parse JSON/XML an toàn (chặn XXE), global prefix `/api/v1` và CORS. | `req.ip` chính xác, `req.id` UUIDv7, `req.body` JS object |
| 9 | BE — Guard | [`app.module.ts`](../../../backend_rag/src/app.module.ts#L32-L47) & [`throttler.ts`](../../../backend_rag/src/common/security/throttler.ts#L16-L46) (`RATE_LIMITS.login`) | `req.ip`, metadata `@RateLimit('login')` | `ThrottlerGuard` toàn cục áp ngưỡng IP cho route đăng nhập: tối đa **`10` request / `60_000ms`**. Vượt ngưỡng ném `429 Too Many Requests` (`Retry-After`). Route công khai không gắn `JwtAuthGuard`. | Cho phép đi tiếp hoặc ngắt `429` |
| 10 | BE — Interceptor chiều vào | [`app.setup.ts`](../../../backend_rag/src/app.setup.ts#L78-L81), [`xml.ts`](../../../backend_rag/src/common/http/xml.ts#L109-L119), [`hateoas.ts`](../../../backend_rag/src/common/http/hateoas.ts#L157-L166) | Header `Accept`, metadata `@WithLinks` | `ContentNegotiationInterceptor` kiểm tra `req.accepts(OFFERED)` (ném `406 NotAcceptableException` nếu không chấp nhận JSON/XML/SSE); `LinksInterceptor` đọc metadata `@WithLinks({ resource: accountLinks })` chuẩn bị cho chiều ra. | Request vượt qua tiền kiểm tra định dạng |
| 11 | BE — Pipe | [`validation.ts`](../../../backend_rag/src/common/http/validation.ts#L18-L26) (`createValidationPipe`) & [`auth.dto.ts`](../../../backend_rag/src/modules/auth/dto/auth.dto.ts#L36-L47) (`LoginDto`) | `req.body` thô | `ValidationPipe` (`whitelist: true`, `forbidNonWhitelisted: true`, `transform: true`) từ chối field thừa, trim `email`, kiểm tra `email` (`<= 254` ký tự, `'invalid email'`) và `password` (`<= 200` ký tự, `'invalid credentials'`). | Thực thể `dto: LoginDto` sạch sẽ |
| 12 | BE — Controller | [`auth.controller.ts`](../../../backend_rag/src/modules/auth/auth.controller.ts#L33-L41) (`AuthController.login`) | `@Body() dto: LoginDto`, `@Ip() ip: string` | Route `@Post('tokens')` trích xuất IP đã chuẩn hóa qua `trust proxy` và chuyển tiếp xuống `this.auth.login(dto, ip)`. | Ủy quyền `Promise<AuthResponseDto>` cho `AuthService` |
| 13 | BE — Service | [`auth.service.ts`](../../../backend_rag/src/modules/auth/auth.service.ts#L119-L183) & [`token.service.ts`](../../../backend_rag/src/modules/auth/token.service.ts#L43-L78) | `dto: LoginDto`, `ip: string` | Chuẩn hóa `email` & `key = sha256(email\|ip)`; tra `attemptCountCache` & `userByEmailCache`; nếu cache miss gọi Repository song song; kiểm tra lockout (`>= 5` lần sai → `429`); luôn chạy `argon2.verify(user ? hash : dummyHash(), password)` chống timing leak; kiểm tra `is_active`; chuẩn bị `updateLastLogin`, ký JWT (`15m`), sinh raw refresh token + `HMAC-SHA256`, xóa lỗi cũ nếu cần. | Yêu cầu đọc/ghi từ các Repository tới MongoDB |
| 14 | BE — Repository | [`login-attempts.repository.ts`](../../../backend_rag/src/modules/auth/repositories/login-attempts.repository.ts#L15-L25), [`users.repository.ts`](../../../backend_rag/src/modules/auth/repositories/users.repository.ts#L29-L39), [`refresh-tokens.repository.ts`](../../../backend_rag/src/modules/auth/repositories/refresh-tokens.repository.ts#L22-L24) | `key`, `email`, `user._id`, `CreateRefreshTokenRecord` | `LoginAttemptsRepository.countByKey` & `UsersRepository.findByEmailLean` (`.lean()`) chạy song song `Promise.all` (1 RTT khi cache miss); nếu sai mật khẩu ghi `attempts.record(key)`; nếu đúng chạy `users.updateLastLogin`, `refreshTokens.create` và `attempts.clearByKey`. | Truy vấn & ghi Mongoose gửi qua tunnel |
| 15 | Database | [`login-attempt.schema.ts`](../../../backend_rag/src/modules/auth/schemas/login-attempt.schema.ts#L4-L17), [`user.schema.ts`](../../../backend_rag/src/modules/auth/schemas/user.schema.ts#L5-L45), [`refresh-token.schema.ts`](../../../backend_rag/src/modules/auth/schemas/refresh-token.schema.ts#L4-L27) | Lệnh đọc/ghi từ Mongoose qua Cloudflare TCP Tunnel | MongoDB thực thi truy vấn & ghi trên 3 collection: `login_attempts` (TTL 15 phút), `users`, và `refresh_tokens` (TTL 7 ngày). | Document `.lean()` & xác nhận ghi |

---

### 3.3. Giai đoạn 3 — Chiều về Backend: Database → BE → HTTP Response

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 16 | Database → BE — Repository | [`users.repository.ts`](../../../backend_rag/src/modules/auth/repositories/users.repository.ts#L29-L39), [`refresh-tokens.repository.ts`](../../../backend_rag/src/modules/auth/repositories/refresh-tokens.repository.ts#L22-L24), [`login-attempts.repository.ts`](../../../backend_rag/src/modules/auth/repositories/login-attempts.repository.ts#L15-L25) | Kết quả đọc/ghi từ MongoDB | Trả `LeanUser`, `attemptCount`, và hoàn tất xác nhận ghi `last_login_at`, chèn `refresh_tokens` (chỉ chứa `token_hash`), dọn `login_attempts` về tầng service. | Plain objects `.lean()` & `Promise<void>` |
| 17 | BE — Service | [`token.service.ts`](../../../backend_rag/src/modules/auth/token.service.ts#L72-L77) & [`auth.service.ts`](../../../backend_rag/src/modules/auth/auth.service.ts#L178-L183) (`AuthService.respond`) | `access_token`, raw `refresh_token`, `user: LeanUser` | Làm ấm `userCache` (`300s`) và `tokenVersionCache` (`5s`); đóng gói `IssuedTokens` (`token_type: 'Bearer'`, `expires_in: 900`) với `UserDto` (loại bỏ `password_hash`); ghi `logEvent('LOGIN_OK', { user_id })`. | `AuthResponseDto` hoàn chỉnh |
| 18 | BE — Controller | [`auth.controller.ts`](../../../backend_rag/src/modules/auth/auth.controller.ts#L33-L41) (`AuthController.login`) | `Promise<AuthResponseDto>` | Trả trực tiếp `AuthResponseDto` ra pipeline NestJS với HTTP status mặc định của `@Post` là `201 Created`. | Payload thô đi qua Interceptor chiều ra |
| 19 | BE — Interceptor chiều ra | [`hateoas.ts`](../../../backend_rag/src/common/http/hateoas.ts#L31-L38) (`accountLinks`, `LinksInterceptor`) & [`xml.ts`](../../../backend_rag/src/common/http/xml.ts#L119-L127) (`ContentNegotiationInterceptor`) | `AuthResponseDto` | `LinksInterceptor` chạy trước ở chiều ra, tạo bản sao nông và chèn `_links: accountLinks()` (`self`, `update`, `password`, `chat-sessions`, `token-rotations`, `token-revocations`). Tiếp theo `ContentNegotiationInterceptor` giữ JSON hoặc chuyển `application/xml` qua `toXml(body)`. | Response body đã gắn `_links` theo đúng `Accept` |
| 20 | BE — NestJS/Express gửi response | [`app.setup.ts`](../../../backend_rag/src/app.setup.ts#L52-L63), [`prometheus-metrics.ts`](../../../backend_rag/src/common/metrics/prometheus-metrics.ts#L57-L76), [`http-exception.filter.ts`](../../../backend_rag/src/common/http/http-exception.filter.ts#L69-L97) | Body JSON/XML hoặc `HttpException` | Express ghi status `201 Created` (hoặc `HttpExceptionFilter` chuẩn hóa lỗi `400`/`401`/`429` thành `{ statusCode, error, message, request_id }`), đính `X-Request-Id`, security headers, CORS `exposedHeaders` và chốt histogram độ trễ khi `finish`. | HTTP Response gửi tới trình duyệt |

#### Wire Example — HTTP `201 Created` Response

```http
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
X-Request-Id: 019367a1-4b2e-7c00-9f12-8a4c2d1e0001

{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "kJ8mP2vQ9xL4nR7wT1yB6cF3hN5sD0aZ8eG4uV2iX9o",
  "user": {
    "id": "67458a12c3f90b0012a4e801",
    "email": "researcher@university.edu",
    "display_name": "Minh Trần",
    "roles": ["user"],
    "created_at": "2026-11-01T08:30:00.000Z"
  },
  "_links": {
    "self": { "href": "/api/v1/users/me" },
    "update": { "href": "/api/v1/users/me", "method": "PATCH" },
    "password": { "href": "/api/v1/users/me/password", "method": "PUT" },
    "chat-sessions": { "href": "/api/v1/chat-sessions" },
    "token-rotations": { "href": "/api/v1/auth/token-rotations", "method": "POST" },
    "token-revocations": { "href": "/api/v1/auth/token-revocations", "method": "POST" }
  }
}
```

---

### 3.4. Giai đoạn 4 — Chiều về Frontend: HTTP Response → FE → Client

| Bước | Tầng (Layer) | File & Ký hiệu | Đầu vào (Input) | Xử lý Nghiệp vụ & Kỹ thuật | Đầu ra (Output) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 21 | FE — HTTP Client (kiểm tra status & đọc body) | [`http.ts`](../../../frontend_rag/src/shared/api/http.ts#L15-L17) (`publicFetch`) | HTTP `Response` từ Backend | Nhận `Response` nguyên vẹn từ `fetch()` (mã `201`, `401`, `429`...) và chuyển ngay về `authApi` mà không kích hoạt auto-rotate 401. | `res: Response` |
| 22 | FE — API (xử lý dữ liệu) | [`authApi.ts`](../../../frontend_rag/src/features/auth/api/authApi.ts#L30-L53) (`readJsonOrThrow`, `loginWithPassword`) | `res: Response` | Nếu `res.ok` (`201`): parse JSON và trả `AuthTokenResponse`. Nếu `!res.ok`: đọc `body?.message` (lấy phần tử đầu nếu là mảng) và ném `AuthApiError(res.status, ..., serverMessage)`. | `AuthTokenResponse` hoặc ném `AuthApiError` |
| 23 | FE — Service | [`authService.ts`](../../../frontend_rag/src/features/auth/services/authService.ts#L20-L42) (`authenticate`, `mapAuthError`) | `AuthTokenResponse` hoặc `unknown` error | Nếu thành công: trả `AuthTokenResponse` về store. Nếu bắt lỗi: `mapAuthError` dịch `401` → `'Email hoặc mật khẩu không chính xác.'`, `429` → `'Quá nhiều lần thử đăng nhập. Vui lòng thử lại sau ít phút.'` (hoặc ánh xạ `serverMessage` / `HttpError`), rồi ném `Error`. | `AuthTokenResponse` hoặc `Error` tiếng Việt |
| 24 | FE — Hook (cập nhật state) | [`authStore.ts`](../../../frontend_rag/src/features/auth/authStore.ts#L40-L93) (`useAuth.login`, `persist`) & [`useAuth.ts`](../../../frontend_rag/src/features/auth/hooks/useAuth.ts#L4-L22) | `data: AuthTokenResponse` | Khi thành công, cập nhật `{ user, accessToken, refreshToken, isAuthenticated: true }` và tự động persist xuống `localStorage` (`'rag-auth-storage'`). Hook `useAuthSession()` phát state mới làm re-render các component tiêu thụ. | State phiên đăng nhập lưu bền vững & phát tới UI |
| 25 | FE — Component (hiển thị lại) | [`AuthOverlay.tsx`](../../../frontend_rag/src/features/auth/AuthOverlay.tsx#L25-L75) & [`ChatPanel.tsx`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L27-L118) | State mới từ `useAuthSession()`, `error` | `finally` tắt `loading = false`; nếu lỗi, `AuthOverlay` gọi `setError(err.message)` hiện banner `AlertTriangle` (`role="alert"`). Nếu thành công, `ChatPanel` unmount `AuthOverlay`, render badge user + `<PipelineStatus />` + `<MessageList />` + `<ChatInput />` và chạy `useEffect` gọi `initSessions()`. | Giao diện chuyển sang trạng thái đã xác thực |
| 26 | Client | [`ChatPanel.tsx`](../../../frontend_rag/src/features/chat/components/ChatPanel.tsx#L58-L118) | Giao diện chat đã đăng nhập hoặc banner lỗi | Người dùng nhìn thấy phiên chat và lịch sử từ IndexedDB/Backend sẵn sàng cho câu hỏi khoa học tiếp theo (hoặc thông báo lỗi cụ thể để nhập lại). | Trải nghiệm người dùng hoàn tất |

---

## 4. Xử lý Lỗi & Các Bất biến Hệ thống (Error Paths & System Invariants)

### 4.1. Bảng Tổng hợp các Đường Lỗi (Error Paths)

| Điều kiện kích hoạt | Lớp phát hiện | HTTP Status & Payload Backend | Ánh xạ & Hiển thị Frontend (`mapAuthError` / `AuthOverlay`) |
| :--- | :--- | :--- | :--- |
| `Content-Type` không phải JSON/XML (khi có body) | [`app.setup.ts`](../../../backend_rag/src/app.setup.ts#L64-L67), [`xml.ts`](../../../backend_rag/src/common/http/xml.ts#L70-L84) | `415 Unsupported Media Type` — `'content type must be application/json or application/xml'` | Ném `AuthApiError(415, ...)` → hiển thị `serverMessage` hoặc fallback |
| XML lỗi cú pháp / chứa `<!DOCTYPE` hoặc `<!ENTITY` | [`xml.ts`](../../../backend_rag/src/common/http/xml.ts#L48-L67) | `400 Bad Request` — `'invalid xml'` | Ném `AuthApiError(400, ...)` → hiển thị `serverMessage` |
| Trường `email` / `password` sai kiểu, thừa trường hoặc quá dài | [`validation.ts`](../../../backend_rag/src/common/http/validation.ts#L18-L25), [`auth.dto.ts`](../../../backend_rag/src/modules/auth/dto/auth.dto.ts#L36-L47) | `400 Bad Request` — `'invalid email'` hoặc `'invalid credentials'` | [`mapAuthError`](../../../frontend_rag/src/features/auth/services/authService.ts#L20-L34) map `'invalid email'` → `'Địa chỉ email không hợp lệ.'`, còn lại dùng `serverMessage` |
| Vượt ngưỡng IP Rate Limit (`> 10` request trong `60s`) | [`throttler.ts`](../../../backend_rag/src/common/security/throttler.ts#L19-L46), [`app.module.ts`](../../../backend_rag/src/app.module.ts#L46) | `429 Too Many Requests` (kèm header `Retry-After` được CORS expose) | `mapAuthError` (`status === 429`) → `'Quá nhiều lần thử đăng nhập. Vui lòng thử lại sau ít phút.'` |
| Sai mật khẩu quá ngưỡng khóa (`attemptCount >= 5` theo `sha256(email\|ip)` trong 15 phút) | [`auth.service.ts`](../../../backend_rag/src/modules/auth/auth.service.ts#L141-L150) | `429 Too Many Requests` — `'too many login attempts'` (log `LOGIN_FAILED`, `reason: 'rate_limited'`) | `mapAuthError` (`status === 429`) → `'Quá nhiều lần thử đăng nhập. Vui lòng thử lại sau ít phút.'` |
| Email không tồn tại hoặc sai mật khẩu (`!user \|\| !ok`) | [`auth.service.ts`](../../../backend_rag/src/modules/auth/auth.service.ts#L158-L163) | `401 Unauthorized` — `'invalid credentials'` (sau khi chạy xong `Argon2id` + ghi `login_attempts`) | `mapAuthError` (`status === 401`) → `'Email hoặc mật khẩu không chính xác.'` |
| Tài khoản bị khóa bởi quản trị viên (`user.is_active === false`) | [`auth.service.ts`](../../../backend_rag/src/modules/auth/auth.service.ts#L164-L170) | `401 Unauthorized` — `'invalid credentials'` (log `reason: 'account_disabled'`, ẩn lý do thực với client) | `mapAuthError` (`status === 401`) → `'Email hoặc mật khẩu không chính xác.'` |
| Client gửi `Accept` không tương thích JSON/XML/SSE | [`xml.ts`](../../../backend_rag/src/common/http/xml.ts#L114-L118) | `406 Not Acceptable` — `'accept must allow application/json or application/xml'` | Ném `AuthApiError(406, ...)` → hiển thị `serverMessage` |
| Lỗi hệ thống / DB mất kết nối không phải `HttpException` | [`http-exception.filter.ts`](../../../backend_rag/src/common/http/http-exception.filter.ts#L32-L50) | `500 Internal Server Error` — `'internal server error'` (che stack trace, gắn `res.err` cho log) | `AuthOverlay` hiển thị `'internal server error'` hoặc thông điệp fallback |

### 4.2. Các Bất biến Hệ thống Cần Bảo toàn (System Invariants)

1. **Chống Brute-Force Hai Tầng & Cấu hình `trust proxy`**:
   - Tầng 1 (`@RateLimit('login')` tại [`throttler.ts`](../../../backend_rag/src/common/security/throttler.ts#L19-L20)): giới hạn **`10` yêu cầu/phút/IP** nhằm chặn credential stuffing diện rộng.
   - Tầng 2 (`MAX_LOGIN_FAILURES = 5` tại [`auth.service.ts`](../../../backend_rag/src/modules/auth/auth.service.ts#L29), [`login-attempt.schema.ts`](../../../backend_rag/src/modules/auth/schemas/login-attempt.schema.ts#L4-L17)): khóa cặp `(email, ip)` được băm thành `sha256(\`${email}\|${ip}\`)` trong cửa sổ trượt **15 phút** (MongoDB TTL `LOGIN_WINDOW_SECONDS = 900` và `attemptCountCache` TTL `15 * 60 * 1000ms`).
   - Bất biến đi kèm: `TRUST_PROXY_HOPS` trong [`security.ts`](../../../backend_rag/src/common/security/security.ts#L46-L52) phải khớp chính xác số lượng reverse proxy thực tế; cấu hình sai cho phép giả mạo `X-Forwarded-For` để qua mặt cả hai tầng bảo vệ.
2. **Chống Dò Tìm Tài khoản qua Thời gian (`dummyHash` Anti-Timing Leak)**:
   - Tại [`auth.service.ts`](../../../backend_rag/src/modules/auth/auth.service.ts#L76-L94) và [`auth.service.ts`](../../../backend_rag/src/modules/auth/auth.service.ts#L152-L157), `argon2.verify` **luôn luôn** được thực thi kể cả khi `user === null`, sử dụng `this.dummyHash()` được sinh lười một lần với cùng cấu hình `ARGON2_OPTIONS` (`argon2id`, `memoryCost: 65536`, `timeCost: 3`, `parallelism: 4`).
   - Tài khoản bị khóa (`is_active: false`) cũng trả về đúng mã `401` với thông điệp `'invalid credentials'` giống hệt sai mật khẩu để không làm lộ trạng thái tồn tại của email.
3. **Thông số `Argon2id` & Threadpool (`UV_THREADPOOL_SIZE = 4`)**:
   - Thông số `ARGON2_OPTIONS` (`64 MB`, `timeCost: 3`, `parallelism: 4`) tại [`auth.service.ts`](../../../backend_rag/src/modules/auth/auth.service.ts#L43-L48) là bất biến bảo mật (`D-021`, `D-022`), kết hợp với `UV_THREADPOOL_SIZE = 4` đặt ở dòng 1 của [`main.ts`](../../../backend_rag/src/main.ts#L1) để tránh nghẽn event loop và các tác vụ I/O khác khi nhiều request đăng nhập chạy đồng thời.
4. **Chiến lược Bộ đệm `LRUCache` & Giảm RTT Cloudflare Tunnel**:
   - Bốn bộ đệm trong [`AuthService`](../../../backend_rag/src/modules/auth/auth.service.ts#L52-L74) đảm nhiệm các vai trò tách biệt:
     - `attemptCountCache` (`max: 10_000`, `ttl: 15m`, không gia hạn trượt): giữ bộ đếm thất bại theo `sha256(email|ip)`.
     - `userByEmailCache` (`max: 5_000`, `ttl: 300s`, `updateAgeOnGet: true`): phục vụ tra cứu nhanh theo `email` lúc đăng nhập.
     - `userCache` (`max: 5_000`, `ttl: 300s`, `updateAgeOnGet: true`): giữ `UserDto` cho `/users/me` và `refresh`.
     - `tokenVersionCache` (`max: 5_000`, `ttl: 5s`, cố ý **không** dùng `updateAgeOnGet`): giúp `JwtAuthGuard` kiểm tra `is_active` và `token_version` trong cửa sổ tối đa 5 giây mà không truy vấn MongoDB mỗi request, đồng thời được nạp trước ngay trong `AuthService.respond` ([`auth.service.ts`](../../../backend_rag/src/modules/auth/auth.service.ts#L314)) để các request ngay sau đăng nhập trúng cache.
   - Khi cache miss ở `login`, `Promise.all` gộp `countByKey` và `findByEmailLean` thành 1 RTT; sau khi xác thực thành công, `respond` và `clearByKey` cũng chạy song song qua `Promise.all`.
5. **Bảo mật Token & Không Lưu Raw Refresh Token ở DB**:
   - Access Token (`JWT`) có thời hạn `15` phút (`ACCESS_TTL_SECONDS = 900`) và mang claim `ver` đồng bộ với `users.token_version` ([`token.service.ts`](../../../backend_rag/src/modules/auth/token.service.ts#L54-L64)).
   - Refresh Token có thời hạn `7` ngày (`REFRESH_TTL_SECONDS = 604800`), sinh ngẫu nhiên 32 bytes `base64url` và chỉ lưu giá trị băm `HMAC-SHA256(raw, JWT_REFRESH_SECRET)` trong collection `refresh_tokens` ([`token.service.ts`](../../../backend_rag/src/modules/auth/token.service.ts#L43-L70)), kèm `family_id` (`UUIDv4`) để thu hồi dây chuyền nếu phát hiện tái sử dụng token.
6. **Chuẩn Hóa Lỗi Chung (`HttpExceptionFilter`) & Bảo vệ Log (`emailHash`)**:
   - Mọi lỗi HTTP đều đi qua [`HttpExceptionFilter`](../../../backend_rag/src/common/http/http-exception.filter.ts#L69-L97) trả về cấu trúc đồng nhất `{ statusCode, error, message, request_id }` (JSON hoặc XML theo `Accept`).
   - Địa chỉ email thô **không bao giờ** xuất hiện trong log sự kiện; [`emailHash`](../../../backend_rag/src/modules/auth/auth.utils.ts#L10-L15) cắt 16 ký tự hex đầu của `sha256(normalizeEmail(email))` khi ghi `LOGIN_FAILED`.
7. **Khác biệt Độ dài Mật khẩu Giữa Đăng nhập và Đăng ký ở FE/BE**:
   - Lưu ý thực tế trong mã nguồn: input mật khẩu ở [`AuthOverlay.tsx`](../../../frontend_rag/src/features/auth/AuthOverlay.tsx#L134) dùng chung `minLength={8}` trên trình duyệt, trong khi Backend [`LoginDto`](../../../backend_rag/src/modules/auth/dto/auth.dto.ts#L43-L46) không áp đặt độ dài tối thiểu khi đăng nhập (chỉ chặn `@MaxLength(200)`) nhằm vẫn cho phép tài khoản cũ đăng nhập và xử lý thống nhất qua `Argon2id`; chính sách tối thiểu 10 ký tự (`PASSWORD_MIN_LENGTH = 10` tại [`common-passwords.ts`](../../../backend_rag/src/modules/auth/common-passwords.ts#L242)) chỉ áp dụng khi đăng ký (`RegisterDto`) hoặc đổi mật khẩu (`ChangePasswordDto`).
