# Khoi_chat_log.md — MongoDB 3-Collection Schema Design Decision Log

- **Motivation/Background**: Designing the MongoDB document-collection architecture for `users`, `chat_sessions` (`conversations`), and `chat_messages` (`messages`) required resolving data ownership, denormalization, RAG metadata persistence, sorting semantics, TTL soft deletion, and compatibility with the merged `feat/monolith-v1` and `frontend_rag` codebase.
- **Purpose**: Record the architectural decisions, debated trade-offs, rejected alternatives, and human vs. AI attribution from the MongoDB 3-collection schema design session.
- **Overview Pipeline**: Initial 3-collection draft proposal $\rightarrow$ 9-point technical critique & debate $\rightarrow$ iterative schema refinement $\rightarrow$ MongoDB B-Tree/TTL verification $\rightarrow$ Hybrid Schema Specification ([`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md)).
- **Detailed Plan**: Index table of decisions `D-001` through `D-011` followed by immutable 12-line decision blocks adhering to the Decision Log and Markdown conventions.
- **References**: [`agents/rules/MD_CONVENTION.md`](../../agents/rules/MD_CONVENTION.md), [`agents/rules/COMMIT_CONVENTION.md`](../../agents/rules/COMMIT_CONVENTION.md), [`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md), [`docs/CHAT_SESSIONS_SPECIFICATION.md`](../CHAT_SESSIONS_SPECIFICATION.md), [`docs/DECISION_LOG.md`](../DECISION_LOG.md).
- **Created**: 2026-10-22T17:29:36+07:00
- **Last Updated**: 2026-10-28T16:40:00+07:00

---

[STATUS: ACTIVE]

## Index

| ID | Title | Decided by | Status |
| :--- | :--- | :--- | :--- |
| D-001 | Allow dual authentication via either `username` or `email` in `users` | human | accepted |
| D-002 | Remove denormalized `title` and unkeyed `check_sum` from `messages` | agent-proposed, human-approved | accepted |
| D-003 | Split `updated_at` into `last_message_at` and `updated_at` in `conversations` | agent-proposed, human-approved | accepted |
| D-004 | Scope `conversations` strictly to authenticated `user_id` without `device_id` | human | accepted |
| D-005 | Preserve RAG synthesis metadata (`citations`, `grounded_chunks`, `timings`) on assistant messages | agent-proposed, human-approved | accepted |
| D-006 | Use `UUIDv7` as primary key (`_id`) for chat sessions and messages | human | accepted |
| D-007 | Cascade 3-day TTL soft deletion (`deleted_at`) across both `chat_sessions` and `chat_messages` | agent-proposed, human-approved | accepted |
| D-008 | Format default session title as `YY-MM-DD:HH-MM-SS` (UTC+07) without first-question overwrite | human | accepted |
| D-009 | Create chat sessions lazily on first message and never store `message_count = 0` sessions | human | accepted |
| D-010 | Enforce fixed 10-item cursor pagination with `has_more` on `GET /api/v1/chat-sessions` | agent-proposed, human-approved | accepted |
| D-011 | Comprehensive test coverage for boundary conditions, query injection resistance, and route auth guards | human | accepted |

---

### D-001 — Allow dual authentication via either `username` or `email` in `users`
- **Status:** accepted
- **When:** 2026-10-22T16:59:06+07:00
- **Decided by:** human
- **Trigger:** "Đăng nhập bằng cả username hoặc email đều được" (Turn 3)
- **Decision:** The `users` collection enforces uniqueness on both `email` (`unique = true`) and `username` (`unique = true`), allowing login with either identifier while preserving `display_name` (`full_name`).
- **Why:** Provide flexible user login via username or email while maintaining unique account identity.
- **Rejected:** Email-only login (existing [`user.schema.ts`](../../backend_rag/src/auth/schemas/user.schema.ts)) — overridden by human requirement; unindexed `username` — rejected due to duplicate account collision risk.
- **Touches:** [`backend_rag/src/auth/schemas/user.schema.ts`](../../backend_rag/src/auth/schemas/user.schema.ts), [`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md)
- **Evidence:** not verified (deferred to separate `Auth`/`Users` implementation step)
- **Revisit if:** External OAuth/SSO providers replace password-based authentication.

### D-002 — Remove denormalized `title` and unkeyed `check_sum` from `messages`
- **Status:** accepted
- **When:** 2026-10-22T16:59:06+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "- messages (Tin nhắn) | id | role | content | session_id | created_at | updated_at |" (Turn 3, dropping `title` and `check_sum` after agent critique)
- **Decision:** Omit `title` and `check_sum` (`SHA256(content + updated_at)`) from `chat_messages` (`messages`); store conversation `title` exclusively in `chat_sessions`.
- **Why:** Embedding `title` in every message causes $O(N)$ write amplification (`updateMany`) on session rename; hashing `content + updated_at` fails deduplication because `updated_at` changes per request and offers no tamper security without an HMAC secret.
- **Rejected:** Embedding `title` and `check_sum` in every message document (Turn 2 draft) — rejected due to write amplification, data inconsistency risk, and ineffective deduplication.
- **Touches:** [`backend_rag/src/chat-sessions/schemas/chat-message.schema.ts`](../../backend_rag/src/chat-sessions/schemas/chat-message.schema.ts), [`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md)
- **Evidence:** `npm --prefix backend_rag test` — 14 suites (150 tests) passed; `npm --prefix backend_rag run test:e2e` — 6 tests passed.

### D-003 — Split `updated_at` into `last_message_at` and `updated_at` in `conversations`
- **Status:** accepted
- **When:** 2026-10-22T17:05:12+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "1. Split up the fields updated_at into last_message_at and updated_at" (Turn 4)
- **Decision:** Maintain two separate timestamps on `chat_sessions`: `last_message_at` (updated only when a message is sent, driving UI session list order) and `updated_at` (updated on any mutation including title rename).
- **Why:** A single `updated_at` field overwrites the last message timestamp when a session is renamed, making it impossible for backend cursor pagination or the UI to keep renamed sessions in their message-activity order.
- **Rejected:** Single `updated_at` field for both rename tracking and message-activity sorting (Turn 2–3 drafts & existing [`chat-session.schema.ts`](../../backend_rag/src/chat-sessions/schemas/chat-session.schema.ts)) — reorders sidebar list on rename.
- **Touches:** [`backend_rag/src/chat-sessions/schemas/chat-session.schema.ts`](../../backend_rag/src/chat-sessions/schemas/chat-session.schema.ts), [`backend_rag/src/chat-sessions/chat-sessions.service.ts`](../../backend_rag/src/chat-sessions/chat-sessions.service.ts), [`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md)
- **Evidence:** `chat-sessions.spec.ts` ("PATCH updates the title and updated_at while preserving last_message_at and list order") — passed.

### D-004 — Scope `conversations` strictly to authenticated `user_id` without `device_id`
- **Status:** accepted
- **When:** 2026-10-22T17:05:12+07:00
- **Decided by:** human
- **Trigger:** "2. Add user_id, don't add device_id, I dont wanna make it complicated at first step right now" (Turn 4)
- **Decision:** Require `user_id` (`ObjectId` referencing `users._id`) on all `chat_sessions` documents and exclude anonymous `device_id` cookie scoping.
- **Why:** Enforces per-user conversation isolation while avoiding anonymous cookie management and guest-to-account session merging complexity in the initial release.
- **Rejected:** Omitting ownership key on `conversations` (Turn 2–3 drafts) — leaks sessions across all users; anonymous `device_id` cookie ([`docs/CHAT_SESSIONS_SPECIFICATION.md`](../CHAT_SESSIONS_SPECIFICATION.md)) — rejected by human to keep the first step simple.
- **Touches:** [`backend_rag/src/chat-sessions/schemas/chat-session.schema.ts`](../../backend_rag/src/chat-sessions/schemas/chat-session.schema.ts), [`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md)
- **Evidence:** `chat-sessions.spec.ts` ("cross-user GET / PATCH / DELETE all return 404 session not found") — passed.
- **Revisit if:** Anonymous guest conversations are required prior to authentication.

### D-005 — Preserve RAG synthesis metadata (`citations`, `grounded_chunks`, `timings`) on assistant messages
- **Status:** accepted
- **When:** 2026-10-22T17:05:12+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "3. Yes, in case the role is AI assistant" (Turn 4)
- **Decision:** Store optional `citations: string[]`, `grounded_chunks: GroundedChunk[]`, and `timings: Timings` embedded fields on `chat_messages` documents when `role === 'assistant'`.
- **Why:** Reopening a saved conversation or reloading the page requires full retrieval provenance to restore the Grounded Inspector, paper citation chips, and 3D Canopy highlights.
- **Rejected:** Storing only `content: string` in `messages` (Turn 2–3 drafts) — permanently discards citations, chunk scores, and pipeline timings after reload.
- **Touches:** [`backend_rag/src/chat-sessions/schemas/chat-message.schema.ts`](../../backend_rag/src/chat-sessions/schemas/chat-message.schema.ts), [`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md)
- **Evidence:** `chat-sessions.spec.ts` ("GET /:id returns messages in chronological order with assistant metadata") — passed.

### D-006 — Use `UUIDv7` as primary key (`_id`) for chat sessions and messages
- **Status:** accepted
- **When:** 2026-10-22T17:05:12+07:00
- **Decided by:** human
- **Trigger:** "4. uuid is required for chat sessions" (Turn 4)
- **Decision:** Override MongoDB `_id: string` with time-ordered `UUIDv7` ([`backend_rag/src/common/uuid.ts`](../../backend_rag/src/common/uuid.ts)) on `chat_sessions` and `chat_messages` (with `session_id` storing the `UUIDv7` string).
- **Why:** Meets the `UUIDv7` session identifier requirement while preserving monotonic B-Tree index insertion order and avoiding a redundant secondary unique index alongside `_id`.
- **Rejected:** Separate `id: uuidv7` field alongside default `_id: ObjectId` — doubles primary-key index RAM overhead; formatted timestamp string as ID — collision and clock-skew risk.
- **Touches:** [`backend_rag/src/chat-sessions/schemas/chat-session.schema.ts`](../../backend_rag/src/chat-sessions/schemas/chat-session.schema.ts), [`backend_rag/src/chat-sessions/schemas/chat-message.schema.ts`](../../backend_rag/src/chat-sessions/schemas/chat-message.schema.ts), [`backend_rag/src/rag/dto/chat-request.dto.ts`](../../backend_rag/src/rag/dto/chat-request.dto.ts), [`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md)
- **Evidence:** `chat-sessions.spec.ts` and `rag.spec.ts` — 35 tests passed with strict `UUIDv7` validation.

### D-007 — Cascade 3-day TTL soft deletion (`deleted_at`) across both `chat_sessions` and `chat_messages`
- **Status:** accepted
- **When:** 2026-10-22T17:05:12+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "deleted_at | Date | Lưu ngày mà user bấm xóa (trong 3 ngày,...)" (Turn 3)
- **Decision:** Add `deleted_at: Date | null` with a 3-day TTL index (`expireAfterSeconds: 259200`) to both `chat_sessions` and `chat_messages`, and set `deleted_at = now` on both collections during `DELETE /api/v1/chat-sessions/:id`.
- **Why:** MongoDB TTL indexes do not cascade across collections; marking both parent session and child messages with `deleted_at` prevents orphaned messages after the 3-day retention window expires.
- **Rejected:** Immediate physical `deleteOne`/`deleteMany` — loses 3-day recovery window; `deleted_at` on `chat_sessions` only — leaves orphaned documents in `chat_messages`.
- **Touches:** [`backend_rag/src/chat-sessions/schemas/chat-session.schema.ts`](../../backend_rag/src/chat-sessions/schemas/chat-session.schema.ts), [`backend_rag/src/chat-sessions/schemas/chat-message.schema.ts`](../../backend_rag/src/chat-sessions/schemas/chat-message.schema.ts), [`backend_rag/src/chat-sessions/chat-sessions.service.ts`](../../backend_rag/src/chat-sessions/chat-sessions.service.ts)
- **Evidence:** `chat-sessions.spec.ts` ("DELETE soft-deletes the session and its messages with deleted_at tombstones") — passed.

### D-008 — Format default session title as `YY-MM-DD:HH-MM-SS` (UTC+07) without first-question overwrite
- **Status:** accepted
- **When:** 2026-10-28T10:25:00+07:00
- **Decided by:** human
- **Trigger:** "Use timestamp format (e.g. YY-MM-DD:HH-MM-SS) as default title until manually renamed" (Turn 10 clarification)
- **Decision:** Initialize `chat_sessions.title` on creation or auto-created exchange using the `YY-MM-DD:HH-MM-SS` timestamp in `UTC+07` (`formatTimestampTitle()`), and remove `title_is_default` auto-overwriting on the first user question.
- **Why:** Provides a deterministic, human-readable chronological session label from creation until the user explicitly renames the session via `PATCH /api/v1/chat-sessions/:id`.
- **Rejected:** `"New conversation"` placeholder overwritten by the first 60 characters of the first user question — rejected by human in favor of the timestamp default title.
- **Touches:** [`backend_rag/src/chat-sessions/schemas/chat-session.schema.ts`](../../backend_rag/src/chat-sessions/schemas/chat-session.schema.ts), [`backend_rag/src/chat-sessions/chat-sessions.service.ts`](../../backend_rag/src/chat-sessions/chat-sessions.service.ts), [`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md)
- **Evidence:** `chat-sessions.spec.ts` and `rag.spec.ts` — verified `/^\d{2}-\d{2}-\d{2}:\d{2}-\d{2}-\d{2}$/` default title creation and preservation across exchanges.

### D-009 — Create chat sessions lazily on first message and never store `message_count = 0` sessions
- **Status:** accepted
- **When:** 2026-10-28T13:08:17+07:00
- **Decided by:** human
- **Trigger:** "4. If message_count = 0 and user doesn't chat anything on that session,, don't store that conversation" and "Remove POST /api/v1/chat-sessions" (Turns 12–13)
- **Decision:** Remove `POST /api/v1/chat-sessions` (`create()`); create `chat_sessions` documents exclusively inside `saveExchange()` / `writeExchange()` with `message_count: 2` after inserting the initial user/assistant message pair, and filter `message_count: { $gt: 0 }` in `list()`.
- **Why:** Prevents polluting MongoDB with empty `message_count = 0` sessions when a user opens the app or starts a new tab without sending a question.
- **Rejected:** Explicit `POST /api/v1/chat-sessions` empty-session creation endpoint — stores unused `message_count = 0` documents.
- **Touches:** [`backend_rag/src/chat-sessions/chat-sessions.controller.ts`](../../backend_rag/src/chat-sessions/chat-sessions.controller.ts), [`backend_rag/src/chat-sessions/chat-sessions.service.ts`](../../backend_rag/src/chat-sessions/chat-sessions.service.ts), [`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md)
- **Evidence:** `chat-sessions.spec.ts` ("does not expose POST /api/v1/chat-sessions" & "excluding any message_count = 0 documents") — passed.

### D-010 — Enforce fixed 10-item cursor pagination with `has_more` on `GET /api/v1/chat-sessions`
- **Status:** accepted
- **When:** 2026-10-28T13:08:17+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "Fixed 10 items per page + add has_more: boolean (per CHAT_SESSIONS_SPECIFICATION.md §3.1)" (Turn 13)
- **Decision:** Bound `GET /api/v1/chat-sessions` to a fixed page size of `10` (`SESSION_PAGE_SIZE = 10`, disallowing client `limit` query overrides) and return `{ items, next_cursor, has_more }`.
- **Why:** Aligns the backend session list contract with [`docs/CHAT_SESSIONS_SPECIFICATION.md`](../CHAT_SESSIONS_SPECIFICATION.md) §3.1 and §7.3 for predictable response sizes.
- **Rejected:** Client-configurable `?limit=1..50` (default `20`) without `has_more` — conflicted with [`docs/CHAT_SESSIONS_SPECIFICATION.md`](../CHAT_SESSIONS_SPECIFICATION.md) §3.1.
- **Touches:** [`backend_rag/src/chat-sessions/dto/chat-session.dto.ts`](../../backend_rag/src/chat-sessions/dto/chat-session.dto.ts), [`backend_rag/src/chat-sessions/chat-sessions.service.ts`](../../backend_rag/src/chat-sessions/chat-sessions.service.ts), [`docs/MONGODB_SCHEMA_DESIGN.md`](../MONGODB_SCHEMA_DESIGN.md)
- **Evidence:** `chat-sessions.spec.ts` ("paginates with fixed page size 10 and has_more") and `flow.e2e-spec.ts` — passed.

### D-011 — Comprehensive test coverage for boundary conditions, query injection resistance, and route auth guards
- **Status:** accepted
- **When:** 2026-10-28T16:38:55+07:00
- **Decided by:** human
- **Trigger:** "In 4. Coverage Gaps, 2. if there are 23 sessions, don't need to add 11 sessions test cases, instead, I think we should add cases when user first use this RAG system, it means they don't have any conversations yet (sessions = 0), and in case they have exact 10 session as limit, also the query URL should be checked for injection case... Add both the 1 and 2 spec on coverage gaps and limitations" (Turns 16–17)
- **Decision:** Implement exhaustive test cases in `chat-sessions.spec.ts` covering empty state (`sessions = 0`), exact 10-session page boundary (`has_more: false`, `next_cursor: null`), 200-message truncation cap (`MAX_MESSAGES`), `@MaxLength` query parameter validation (`q > 120`, `cursor > 512`), ReDoS & NoSQL query injection protection, cross-user cursor isolation, and 401 JWT auth guards on all individual endpoints.
- **Why:** Verifies system resilience against injection attacks, boundary off-by-one pagination bugs, payload bloat, and unauthorized access across all chat session routes.
- **Rejected:** 11-session test case — rejected by human as redundant since the 23-session test case already proves multi-page cursor transitions.
- **Touches:** [`backend_rag/src/chat-sessions/chat-sessions.spec.ts`](../../backend_rag/src/chat-sessions/chat-sessions.spec.ts), [`backend_rag/docs/CHAT_SESSIONS-SPEC-TESTS.md`](../../backend_rag/docs/CHAT_SESSIONS-SPEC-TESTS.md)
- **Evidence:** `chat-sessions.spec.ts` — 20 Jest tests (34 distinct scenarios) passed in 3.386 s; 0 lint/build errors.
