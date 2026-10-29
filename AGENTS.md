# AGENTS.md — Root AI Agent Operating Manual & Knowledge Map

- **Motivation/Background**: As `RAG_TS` scaled into a full-stack scientific RAG platform with a 3D React Three Fiber frontend (`frontend_rag`), a high-concurrency NestJS 11 backend (`backend_rag`), an offline-first IndexedDB sync engine, and a 3-stage SRE load-testing suite, AI coding agents needed a single authoritative root entrypoint to navigate the repository without context bloat.
- **Purpose**: Provide AI coding agents with the progressive-disclosure documentation routing map, repository topology, non-negotiable architectural invariants, and workflow rules across the entire `RAG_TS` workspace.
- **Overview Pipeline**: Task Identification → Progressive-Disclosure Doc Lookup (`docs/INDEX.md`) → Subsystem `AGENTS.md` Inspection → Implementation & Invariant Verification → Decision Log & Convention Compliance.
- **Detailed Plan**: Section 1 defines the Progressive-Disclosure Routing Table; Section 2 maps the Monorepo Topology; Section 3 enforces Golden System Invariants; Section 4 lists Essential Verification Commands; Section 5 enforces Commit & Markdown Conventions.
- **References**: [docs/INDEX.md](docs/INDEX.md), [docs/ARCHITECTURE_OVERVIEW.md](docs/ARCHITECTURE_OVERVIEW.md), [backend_rag/AGENTS.md](backend_rag/AGENTS.md), [frontend_rag/AGENTS.md](frontend_rag/AGENTS.md), [docs/FE_DESIGN_SYSTEM.md](docs/FE_DESIGN_SYSTEM.md), [docs/FE3d_prompt_v2.md](docs/FE3d_prompt_v2.md), [agents/rules/MD_CONVENTION.md](agents/rules/MD_CONVENTION.md), [agents/rules/COMMIT_CONVENTION.md](agents/rules/COMMIT_CONVENTION.md), [docs/AI_chat_log/Vuong_chat_logs.md](docs/AI_chat_log/Vuong_chat_logs.md).
- **Created**: 2026-10-29T19:42:00+07:00
- **Last Updated**: 2026-10-30T06:13:00+07:00

[STATUS: ACTIVE]

---

## 1. Progressive-Disclosure Routing Table (Read Only What You Need)

To avoid flooding your context window, **do not read every `.md` file in the repository**. Start here and open only the documents matching your active task:

| If your task involves... | Read First (Entrypoint) | Deep-Dive Reference(s) |
| :--- | :--- | :--- |
| **Finding which docs are current vs. archived** | [`docs/INDEX.md`](docs/INDEX.md) | [`docs/AI_chat_log/Vuong_chat_logs.md`](docs/AI_chat_log/Vuong_chat_logs.md) |
| **Full-stack architecture, data flow, or API contracts** | [`docs/ARCHITECTURE_OVERVIEW.md`](docs/ARCHITECTURE_OVERVIEW.md) | [`docs/API_SPECIFICATION.md`](docs/API_SPECIFICATION.md), [`docs/API_DESIGN.md`](docs/API_DESIGN.md) |
| **Backend NestJS modules, controllers, or MongoDB schemas** | [`backend_rag/AGENTS.md`](backend_rag/AGENTS.md) | [`backend_rag/docs/MONGODB_SCHEMA_DESIGN.md`](backend_rag/docs/MONGODB_SCHEMA_DESIGN.md), [`docs/SCIENTIFIC_RAG_BACKEND.md`](docs/SCIENTIFIC_RAG_BACKEND.md) |
| **Authentication, JWT, Argon2id, Refresh Rotation, or Logout** | [`backend_rag/docs/AUTH_AND_SECURITY_ARCHITECTURE.md`](backend_rag/docs/AUTH_AND_SECURITY_ARCHITECTURE.md) | [`backend_rag/docs/AUTH_LOGOUT_DESIGN.md`](backend_rag/docs/AUTH_LOGOUT_DESIGN.md) |
| **Chat Sessions, UUIDv7 cursor pagination, or soft-delete** | [`backend_rag/docs/CHAT_SESSIONS_SPECIFICATION.md`](backend_rag/docs/CHAT_SESSIONS_SPECIFICATION.md) | [`backend_rag/docs/CHAT_SESSIONS_SPEC-TESTS.md`](backend_rag/docs/CHAT_SESSIONS_SPEC-TESTS.md) |
| **Backend caching (`LRUCache`), latency optimization, or k6 stress testing** | [`backend_rag/docs/PERFORMANCE_AND_CACHING.md`](backend_rag/docs/PERFORMANCE_AND_CACHING.md) | [`backend_rag/benchmark/BOTTLENECK_ANALYSIS_AND_OPTIMIZATION.md`](backend_rag/benchmark/BOTTLENECK_ANALYSIS_AND_OPTIMIZATION.md), [`backend_rag/benchmark/README.md`](backend_rag/benchmark/README.md) |
| **Frontend 3D World (`Three.js` / `R3F`), UI, or Web Vitals** | [`frontend_rag/AGENTS.md`](frontend_rag/AGENTS.md) | [`frontend_rag/docs/perf-chat-report.md`](frontend_rag/docs/perf-chat-report.md), [`frontend_rag/docs/perf-report.md`](frontend_rag/docs/perf-report.md) |
| **UI design tokens, glass materials, Liquid Glass, or 3D rotation math (`P15`–`P23`)** | [`docs/FE_DESIGN_SYSTEM.md`](docs/FE_DESIGN_SYSTEM.md) | [`docs/FE3d_prompt_v2.md`](docs/FE3d_prompt_v2.md), [`docs/FE_DESIGN_AUDIT.md`](docs/FE_DESIGN_AUDIT.md) |
| **Frontend state (`Zustand`), IndexedDB (`Dexie.js`), or SSE streaming** | [`frontend_rag/docs/STATE_AND_STREAMING_ARCHITECTURE.md`](frontend_rag/docs/STATE_AND_STREAMING_ARCHITECTURE.md) | [`frontend_rag/AGENTS.md`](frontend_rag/AGENTS.md) |
| **Writing `.md` documentation, Decision Logs, or Git commits** | [`agents/rules/MD_CONVENTION.md`](agents/rules/MD_CONVENTION.md) | [`agents/template/DECISION_LOG_TEMPLATE.md`](agents/template/DECISION_LOG_TEMPLATE.md), [`agents/rules/COMMIT_CONVENTION.md`](agents/rules/COMMIT_CONVENTION.md) |

---

## 2. Workspace & Monorepo Topology

```
RAG_TS/                                  # Workspace Root
├── AGENTS.md                            # [YOU ARE HERE] Root AI agent operating manual
├── README.md                            # Human-facing project overview & quickstart
├── agents/                              # Governance rules & templates for AI agents
│   ├── rules/
│   │   ├── COMMIT_CONVENTION.md         # Git commit message format rules
│   │   └── MD_CONVENTION.md             # Mandatory 7-field header & [STATUS] rules for all .md files
│   └── template/
│       └── DECISION_LOG_TEMPLATE.md     # 12-line decision block schema & attribution workflow
├── docs/                                # Cross-cutting architecture specs, reports, and decision logs
│   ├── INDEX.md                         # Master catalog of all .md files (ACTIVE vs ARCHIVED)
│   ├── ARCHITECTURE_OVERVIEW.md         # End-to-end full-stack architecture & request lifecycle
│   ├── API_SPECIFICATION.md             # REST & SSE API endpoint contracts (/api/v1/*)
│   ├── SCIENTIFIC_RAG_BACKEND.md        # Vector retrieval, PageRank authority & FP-Growth mining
│   ├── FE_DESIGN_SYSTEM.md              # Unified UI design tokens, glass materials & 3D rotation rules
│   ├── FE_DESIGN_AUDIT.md               # Empirical UI token drift & Liquid Glass audit
│   ├── FE3d_prompt_v2.md                # Sequential P15-P23 prompt plan for Design System & Liquid Glass
│   ├── RUN_GUIDE.md                     # Local environment startup (Cloudflare tunnel, Qdrant, Ollama)
│   └── AI_chat_log/
│       └── Vuong_chat_logs.md           # Authoritative Decision Log (D-001 through D-022+)
├── backend_rag/                         # NestJS 11 + TypeScript 5.7 Backend Monolith (Port 3000)
│   ├── AGENTS.md                        # Backend-specific AI agent guide & module map
│   ├── src/                             # NestJS modules (auth, chat-sessions, rag, world, vector-store, llm, mining)
│   ├── docs/                            # Backend subsystem specs (Auth, Caching, MongoDB, Chat Sessions)
│   └── benchmark/                       # 3-Stage Production SRE k6 Stress Test Suite & Python Analytics
└── frontend_rag/                        # React 19 + Vite + Three.js (R3F) + Zustand + Dexie.js (Port 5173)
    ├── AGENTS.md                        # Frontend-specific AI agent guide & component map
    ├── src/                             # 3D world canvas, shaders, chat UI, IndexedDB store, API client
    └── docs/                            # Frontend state/streaming architecture & WebGL/Web Vitals reports
```

---

## 3. Golden System Invariants (Never Violate)

1. **Read-Only Vector Store (`Qdrant`)**:
   - `backend_rag` only **reads** from the `scientific_papers_gold` Qdrant collection (auto-detecting `https: true` on port `443` per `D-004`). Never drop, recreate, or mutate Qdrant collections from `backend_rag`.
2. **Cloudflare TCP Tunnel for Remote MongoDB (`D-002`)**:
   - Remote MongoDB (`mongodb.minio-quangmanh.io.vn`) sits behind Cloudflare Access and must be accessed via a local `cloudflared access tcp` daemon bound to `localhost:27018`. Every network round-trip to MongoDB costs `~15–25ms` over the tunnel — **never introduce sequential N+1 MongoDB queries** when `Promise.all` or write-through `LRUCache` can be used.
3. **Cryptographic & Auth Security Invariants (`D-021`, `D-022`)**:
   - Passwords use **`Argon2id`** (`memoryCost: 65536` = `64 MB`, `timeCost: 3`, `parallelism: 4`) with `UV_THREADPOOL_SIZE = 4` in [`backend_rag/src/main.ts`](backend_rag/src/main.ts). Never weaken `Argon2id` parameters.
   - Refresh tokens use **1-RTT optimistic atomic rotation** (`findOneAndUpdate({ token_hash, revoked_at: null })`) in [`backend_rag/src/auth/token.service.ts`](backend_rag/src/auth/token.service.ts) with full `family_id` revocation upon reuse detection.
4. **Generation-Versioned Cache Invalidation (`D-021`)**:
   - [`ChatSessionsService`](backend_rag/src/chat-sessions/chat-sessions.service.ts) caches paginated session lists (`listCache`) and message histories (`sessionDetailCache`) keyed by a per-user generation counter (`userListGen.get(userId)`). Any new endpoint that mutates sessions or messages **must** call `this.bumpUserGen(userId)` (or update the cache in-place) so users never read stale data.
5. **UUIDv7 Primary Keys & Soft-Delete Filtering**:
   - `chat_sessions._id` and `chat_messages._id` use **UUIDv7** strings (time-ordered) generated by the application, never MongoDB `ObjectId`.
   - Every query on `chat_sessions` and `chat_messages` **must** include `deleted_at: null` and enforce ownership (`user_id: userId`), returning `404 Not Found` (never `403 Forbidden`) when a session belongs to another user or is soft-deleted.
6. **Offline-First Frontend Chat Persistence (`D-017`)**:
   - [`frontend_rag/src/db/chatDb.ts`](frontend_rag/src/db/chatDb.ts) (`Dexie.js` IndexedDB) is the primary client-side ground truth so chat history remains 100% viewable and navigable even if `backend_rag` is offline.
7. **3D Render Loop & Streaming Performance (`D-005` – `D-013`)**:
   - Never render raw HTML DOM elements (`<div>`, `<span>`) directly inside R3F `<Canvas>` (`D-013`).
   - Keep `useDeferredValue` and the `30ms` micro-batched streaming buffer in [`frontend_rag/src/store/ragStore.ts`](frontend_rag/src/store/ragStore.ts) so KaTeX/Markdown AST parsing never freezes the WebGL render loop.

---

## 4. Essential Verification Commands

- **Backend Unit & Integration Tests (`backend_rag/`)**:
  ```powershell
  cd backend_rag
  npx jest src/auth/auth.spec.ts src/chat-sessions/chat-sessions.spec.ts --runInBand
  npm test
  ```
- **Backend Production Build & Typecheck (`backend_rag/`)**:
  ```powershell
  cd backend_rag
  npm run build
  ```
- **Backend 3-Stage SRE Stress Test Certification (`backend_rag/benchmark/`)**:
  - Requires `P95 < 500ms`, `P99 < 1000ms`, `Error Rate < 1%` held continuously for **3 minutes (`180s`)** per Rule L1 (`D-019`). See [`backend_rag/docs/PERFORMANCE_AND_CACHING.md`](backend_rag/docs/PERFORMANCE_AND_CACHING.md).
- **Frontend Typecheck, Vitest & Playwright E2E Suite (`frontend_rag/`)**:
  ```powershell
  cd frontend_rag
  npm run typecheck
  npm test -- --run
  npx playwright test e2e/knowledge-tree.spec.ts
  ```

---

## 5. Markdown & Decision Log Governance

1. **All `.md` files** created or updated in this workspace must include the **7-field metadata header** (`Motivation/Background`, `Purpose`, `Overview Pipeline`, `Detailed Plan`, `References`, `Created`, `Last Updated`) and a `[STATUS: DRAFT | IN_PROGRESS | ACTIVE | ARCHIVED]` badge per [`agents/rules/MD_CONVENTION.md`](agents/rules/MD_CONVENTION.md).
2. **All significant architectural, performance, or schema decisions** must be logged in [`docs/AI_chat_log/Vuong_chat_logs.md`](docs/AI_chat_log/Vuong_chat_logs.md) following [`agents/template/DECISION_LOG_TEMPLATE.md`](agents/template/DECISION_LOG_TEMPLATE.md).
3. **All git commits** must follow [`agents/rules/COMMIT_CONVENTION.md`](agents/rules/COMMIT_CONVENTION.md).
