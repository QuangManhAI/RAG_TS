# INDEX.md — Master Documentation Catalog & Status Registry

- **Motivation/Background**: As `RAG_TS` evolved across multiple sprints (initial RAG bootstrapping, 3D WebGL world optimization, Auth/Chat-Sessions MongoDB persistence, Offline-First IndexedDB sync, 3-Stage Production SRE Stress Testing, and the Frontend Design System / Liquid Glass overhaul), over 30 Markdown files accumulated across `docs/`, `backend_rag/`, `frontend_rag/`, and `agents/`. Without a central registry, AI agents risk reading outdated bootstrap prompts instead of active production specifications.
- **Purpose**: Provide a complete, status-tagged catalog (`ACTIVE` vs. `ARCHIVED`) of every Markdown document in the `RAG_TS` monorepo so AI agents and human engineers can locate the authoritative source of truth in seconds.
- **Overview Pipeline**: Domain Classification → File Path & Status Badge Lookup → Progressive-Disclosure Navigation.
- **Detailed Plan**: Files live under `docs/` by topic: `architecture/`, `api/`, `design/`, `guides/`, `security/`, `research/`, `reports/`, `slides/`, `prompts/`, `assets/`.  Section 1 catalogs AI Agent Entrypoints & Governance Rules; Section 2 catalogs System Architecture & API Specifications; Section 3 catalogs Backend Engineering & SRE Benchmark Docs; Section 4 catalogs Frontend 3D, Design System & Offline Persistence Docs; Section 5 lists Archived Bootstrap Prompts (do not use for active code generation).
- **References**: [AGENTS.md](../AGENTS.md), [backend_rag/AGENTS.md](../backend_rag/AGENTS.md), [frontend_rag/AGENTS.md](../frontend_rag/AGENTS.md), [agents/rules/MD_CONVENTION.md](../agents/rules/MD_CONVENTION.md), [docs/AI_chat_log/Vuong_chat_logs.md](AI_chat_log/Vuong_chat_logs.md).
- **Created**: 2026-10-29T19:43:00+07:00
- **Last Updated**: 2026-11-09T17:51:00+07:00

[STATUS: ACTIVE]

---

## 1. AI Agent Entrypoints, Governance Rules & Decision Logs

| Document | Status | Audience & Scope |
| :--- | :---: | :--- |
| [`AGENTS.md`](../AGENTS.md) | `ACTIVE` | **Root AI Agent Entrypoint**: Progressive-disclosure routing table, monorepo structure, golden invariants, and verification commands. |
| [`docs/INDEX.md`](INDEX.md) | `ACTIVE` | **Master Documentation Catalog** *(this file)*: Maps every `.md` file across `RAG_TS/`, `backend_rag/`, `frontend_rag/`, and `agents/`. |
| [`backend_rag/AGENTS.md`](../backend_rag/AGENTS.md) | `ACTIVE` | **Backend AI Agent Entrypoint**: NestJS 11 module inventory, security/caching invariants, MongoDB rules, and test/benchmark workflows. |
| [`frontend_rag/AGENTS.md`](../frontend_rag/AGENTS.md) | `ACTIVE` | **Frontend AI Agent Entrypoint**: React 19 + Three.js (R3F) + Zustand + Dexie.js architecture, 3D rendering rules, and Web Vitals guardrails. |
| [`docs/AI_chat_log/Vuong_chat_logs.md`](AI_chat_log/Vuong_chat_logs.md) | `ACTIVE` | **Authoritative Decision Log (`D-001` – `D-022`)**: Immutable 12-line records of every human and agent engineering decision with empirical evidence. |
| [`agents/rules/MD_CONVENTION.md`](../agents/rules/MD_CONVENTION.md) | `ACTIVE` | **Markdown Standard**: Mandatory 7-field header, ISO 8601 `+07:00` timestamps, and `[STATUS]` lifecycle tags. |
| [`agents/rules/COMMIT_CONVENTION.md`](../agents/rules/COMMIT_CONVENTION.md) | `ACTIVE` | **Git Commit Standard**: Conventional commit prefixes and scope rules. |
| [`agents/template/DECISION_LOG_TEMPLATE.md`](../agents/template/DECISION_LOG_TEMPLATE.md) | `ACTIVE` | **Decision Log Schema**: 12-line block specification, `human` vs. `agent-proposed, human-approved` vs. `agent-default` attribution rules. |

---

## 2. System Architecture, API Contracts & Operations (`docs/`)

| Document | Status | Audience & Scope |
| :--- | :---: | :--- |
| [`docs/architecture/ARCHITECTURE_OVERVIEW.md`](architecture/ARCHITECTURE_OVERVIEW.md) | `ACTIVE` | **Full-Stack System Architecture**: End-to-end topology (`frontend_rag` $\leftrightarrow$ `backend_rag` $\leftrightarrow$ MongoDB / Qdrant / LLM), SSE streaming flow, and dual-persistence model. |
| [`README.md`](../README.md) | `ACTIVE` | **Project Overview**: High-level introduction to the RAG 3D Knowledge Tree system and repository layout. |
| [`docs/guides/RUN_GUIDE.md`](guides/RUN_GUIDE.md) | `ACTIVE` | **Environment & Startup Guide**: How to run `cloudflared` TCP tunnel (`localhost:27018`), Qdrant, Ollama/GGUF model, `backend_rag`, and `frontend_rag`. |
| [`docs/api/API_SPECIFICATION.md`](api/API_SPECIFICATION.md) | `ACTIVE` | **REST & SSE API Reference**: Detailed request/response payloads, status codes, and error envelopes for `/api/v1/*`. |
| [`docs/api/API_DESIGN.md`](api/API_DESIGN.md) | `ACTIVE` | **Core API Design Contract**: Standard error format (`statusCode`, `error`, `message`, `details`), pagination rules, and route conventions. |
| [`docs/prompts/prompt_security.md`](prompts/prompt_security.md) | `ACTIVE` | **Security Hardening & RBAC Specification (`admin`/`user`)**: Source of truth for the S1–S8 security work — security baseline (`helmet`, `trust proxy`, `@nestjs/throttler`), password policy, `token_version` token invalidation, the 13-route admin surface, the append-only `admin_audit_logs` trail, and the admin bootstrap CLI. Includes the implementation report. |
| [`docs/architecture/SCIENTIFIC_RAG_BACKEND.md`](architecture/SCIENTIFIC_RAG_BACKEND.md) | `ACTIVE` | **Scientific RAG Pipeline**: Nomic v1.5 embeddings (`search_query:` prefix), Qdrant cosine search, PageRank authority boost, FP-Growth query expansion, and LLM citation grounding. |
| [`docs/architecture/RAG.md`](architecture/RAG.md) | `ACTIVE` | **RAG Pipeline Concepts & Domain Notes**: Theoretical and practical notes on retrieval, reranking, and prompt grounding. |
| [`docs/reports/REPORT.md`](reports/REPORT.md) | `ACTIVE` | **Executive Engineering & Performance Report**: Consolidated summary of WebGL 3D optimizations, Lighthouse 97 audit, and backend load qualification. |
| [`docs/reports/integrated-bench-run-2026-10-23T08-19-38-jjpmf-executive.md`](reports/integrated-bench-run-2026-10-23T08-19-38-jjpmf-executive.md) | `ACTIVE` | **Integrated Benchmark Run Report**: Historical snapshot of early integrated pipeline benchmark metrics. |
| [`docs/api/API-compare.md`](api/API-compare.md) | `ACTIVE` | **API Before/After Comparison**: The redesign against the 8 API principles. |
| [`docs/architecture/database-v2.md`](architecture/database-v2.md) | `IN_PROGRESS` | **Storage Redesign**: Qdrant, MongoDB and MinIO after moving chunk content out of Qdrant. |
| [`docs/security/agent-sec.md`](security/agent-sec.md) | `IN_PROGRESS` | **Backend Security Review**: Findings and fixes before production. |
| [`docs/research/laya-jev.md`](research/laya-jev.md) | `ACTIVE` | **Laya Evaluation**: Results of adding Laya to the RAG pipeline, with the JEV sources beside it. |
| [`docs/grafana_analysis/GRAFANA_MONITORING_ANALYSIS.md`](grafana_analysis/GRAFANA_MONITORING_ANALYSIS.md) | `ACTIVE` | **Grafana Production Telemetry & SRE Forensic Analysis**: Comparative analysis of the `10:20 AM – 11:40 AM` and `8:00 PM – 8:30 PM` Grafana windows across all 11 panels, bimodal REST vs. SSE latency separation, native C++ RSS pulse profiling, and the `20:20:50` backend restart reconstruction. |
| [`docs/slides/grafana_report_slides.html`](slides/grafana_report_slides.html) | `ACTIVE` | **Grafana SRE Forensic Slide Deck**: 6-slide executive presentation covering scorecard KPIs, comparative matrix table, bimodal REST vs SSE latency graph, 20:20:50 restart reconstruction, C++ memory pulse elasticity, and actionable SRE remediation plan. |
| [`docs/prompts/plan_laya_benchmark_vs_baseline.md`](prompts/plan_laya_benchmark_vs_baseline.md) | `ACTIVE` | **Laya A/B Benchmark Plan**: Implementation plan for benchmarking Laya (20261108-seed0) vs Baseline on `test_set_v1.jsonl` (150 queries) via DeepEval with Local Qwen 2.5-7B Judge on port 9001. |
| [`docs/reports/LAYA_AB_BENCHMARK_REPORT.md`](reports/LAYA_AB_BENCHMARK_REPORT.md) | `ACTIVE` | **Laya A/B Benchmark Report**: Comprehensive 2-tier benchmark report comparing Laya New (`20261108-seed0`) vs Pretrained Baseline across 150 test queries with DeepEval on local Qwen 2.5-7B Judge. |

---

## 3. Backend Engineering, Security, Caching & SRE Benchmarks (`backend_rag/`)

| Document | Status | Audience & Scope |
| :--- | :---: | :--- |
| [`backend_rag/AGENTS.md`](../backend_rag/AGENTS.md) | `ACTIVE` | **Backend AI Manual**: Module directory, DTO/validation rules, cache invalidation checklist, and test commands. |
| [`backend_rag/README.md`](../backend_rag/README.md) | `ACTIVE` | **Backend Service Guide**: Setup, `.env` variables, scripts, and endpoint summary for `backend_rag`. |
| [`backend_rag/docs/AUTH_AND_SECURITY_ARCHITECTURE.md`](../backend_rag/docs/AUTH_AND_SECURITY_ARCHITECTURE.md) | `ACTIVE` | **Auth & Security Deep Dive**: `Argon2id` (`64 MB`, `UV_THREADPOOL_SIZE=4`), JWT access tokens, 1-RTT atomic refresh rotation, token family reuse revocation, progressive lockout, and auth caches. |
| [`backend_rag/docs/AUTH_LOGOUT_DESIGN.md`](../backend_rag/docs/AUTH_LOGOUT_DESIGN.md) | `ACTIVE` | **Logout & Session Revocation Spec**: Single-device `/auth/logout` vs. all-device `/auth/logout-all` token revocation semantics. |
| [`backend_rag/docs/PERFORMANCE_AND_CACHING.md`](../backend_rag/docs/PERFORMANCE_AND_CACHING.md) | `ACTIVE` | **Performance, Caching & SRE Playbook**: Generation-versioned `LRUCache` (`bumpUserGen`), `Singleflight` coalescing, `.lean()` DTO paths, MongoDB pool tuning (`maxPoolSize: 120`), and Rule L1 k6 certification. |
| [`backend_rag/benchmark/BOTTLENECK_ANALYSIS_AND_OPTIMIZATION.md`](../backend_rag/benchmark/BOTTLENECK_ANALYSIS_AND_OPTIMIZATION.md) | `ACTIVE` | **SRE Bottleneck Analysis & `4.0x` Optimization Report**: Root-cause profiling (`100 CUs` $\rightarrow$ `400 CUs` / `250 RPS` certified over 3-min hold-out) and empirical dead-ends (`D-021`, `D-022`). |
| [`backend_rag/benchmark/README.md`](../backend_rag/benchmark/README.md) | `ACTIVE` | **Benchmark Suite Manual**: Complete reference for all k6 scripts (`k6_prod_open_breakpoint.ts`, `k6_prod_certified_holdout.ts`, `k6_prod_auth_crypto.ts`), seeders, and Python chart generators. |
| [`backend_rag/docs/MONGODB_SCHEMA_DESIGN.md`](../backend_rag/docs/MONGODB_SCHEMA_DESIGN.md) | `ACTIVE` | **MongoDB Schema & Index Specification**: Collections (`users`, `refresh_tokens`, `login_attempts`, `chat_sessions`, `chat_messages`), UUIDv7 keys, and compound B-tree indexes. |
| [`backend_rag/docs/CHAT_SESSIONS_SPECIFICATION.md`](../backend_rag/docs/CHAT_SESSIONS_SPECIFICATION.md) | `ACTIVE` | **Chat Sessions Functional Spec**: Cursor-based pagination, soft-delete (`deleted_at`), auto-titling, and ownership isolation rules. |
| [`backend_rag/docs/CHAT_SESSIONS_SPEC-TESTS.md`](../backend_rag/docs/CHAT_SESSIONS_SPEC-TESTS.md) | `ACTIVE` | **Chat Sessions Test Matrix**: Specification of integration test cases in `src/modules/chat-sessions/chat-sessions.spec.ts`. |
| [`backend_rag/ASSUMPTIONS.md`](../backend_rag/ASSUMPTIONS.md) | `ACTIVE` | **Backend Engineering Assumptions Log**: Documented boundary assumptions `[B1]..[B11]` made during backend implementation. |
| [`backend_rag/PROGRESS.md`](../backend_rag/PROGRESS.md) | `ACTIVE` | **Backend Module Implementation Tracker**: Status log of completed NestJS modules and test suites. |

---

## 4. Frontend 3D World, Design System, Offline State & Performance (`frontend_rag/` & `docs/`)

| Document | Status | Audience & Scope |
| :--- | :---: | :--- |
| [`frontend_rag/AGENTS.md`](../frontend_rag/AGENTS.md) | `ACTIVE` | **Frontend AI Manual**: Component map, R3F `<Canvas>` rules, Zustand stores, Dexie.js IndexedDB rules, and Web Vitals guardrails. |
| [`frontend_rag/README.md`](../frontend_rag/README.md) | `ACTIVE` | **Frontend App Guide**: Vite dev server, environment configuration, and build scripts. |
| [`docs/design/FE_DESIGN_SYSTEM.md`](design/FE_DESIGN_SYSTEM.md) | `ACTIVE` | **Frontend Design System & Liquid Glass Spec**: Single source of truth for design tokens (color, radius, blur, glass materials, typography, motion), complex/quaternion 3D rotation math, and Liquid Glass levels 1–4. |
| [`docs/design/FE_DESIGN_AUDIT.md`](design/FE_DESIGN_AUDIT.md) | `ACTIVE` | **Frontend Token Drift & Design Audit**: Empirical measurement of UI token drift across P1–P14 and post-P14 commits prior to `FE_DESIGN_SYSTEM.md`. |
| [`docs/prompts/FE3d_prompt_v2.md`](prompts/FE3d_prompt_v2.md) | `ACTIVE` | **Frontend v2 Prompt Plan (`P15`–`P23`)**: Step-by-step specification, pseudocode, and verification plan for Design System migration, 3D complex/quaternion math, and Liquid Glass. |
| [`frontend_rag/docs/STATE_AND_STREAMING_ARCHITECTURE.md`](../frontend_rag/docs/STATE_AND_STREAMING_ARCHITECTURE.md) | `ACTIVE` | **State, Offline Sync & Streaming Deep Dive**: `ragStore`, `sessionStore`, `Dexie.js` Cache-First persistence, background `/chat-sessions` sync, and `30ms` micro-batched SSE token streaming. |
| [`frontend_rag/docs/perf-chat-report.md`](../frontend_rag/docs/perf-chat-report.md) | `ACTIVE` | **3D Render & Streaming Chat Performance Report**: Empirical analysis of draw-call batching, frustum clipping, DPR clamping, and `PerfHUD` telemetry (`D-005` – `D-013`). |
| [`frontend_rag/docs/perf-report.md`](../frontend_rag/docs/perf-report.md) | `ACTIVE` | **WebGL & Core Web Vitals Audit**: Lighthouse performance optimization (`FCP 0.9s`, `LCP 1.0s`, Score `97/100`). |
| [`frontend_rag/docs/trig-audit.md`](../frontend_rag/docs/trig-audit.md) | `ACTIVE` | **Shader & Geometry Math Audit**: Trigonometric and procedural geometry notes for the 3D knowledge tree. |
| [`frontend_rag/ASSUMPTIONS.md`](../frontend_rag/ASSUMPTIONS.md) | `ACTIVE` | **Frontend Engineering Assumptions Log**: Documented UI/UX and API boundary assumptions. |
| [`frontend_rag/PROGRESS.md`](../frontend_rag/PROGRESS.md) | `ACTIVE` | **Frontend Feature Implementation Tracker**: Module completion and Vitest/Playwright verification log. |
| [`frontend_rag/docs/design-migration-report.md`](../frontend_rag/docs/design-migration-report.md) | `ACTIVE` | **Design Migration Report**: per-prompt `check:design` counts, scale STATS, screenshot diffs and frame-time measurements. |
| [`docs/design/liquid-glass.md`](design/liquid-glass.md) | `ACTIVE` | **Liquid Glass Write-Up**: How the glass of the knowledge tree UI works and why. |

---

## 5. Archived Bootstrap Prompts & Course Notes (`[STATUS: ARCHIVED]`)

> [!WARNING]
> The files below are **historical bootstrapping prompts and early course notes**. They reference early prototypes (e.g., LanceDB instead of Qdrant, or pre-Design-System UI tokens). **AI agents must NOT use these files as specifications when modifying current code.** Always use the `ACTIVE` documents in Sections 1–4 instead.

| Document | Status | Historical Context |
| :--- | :---: | :--- |
| [`docs/prompts/prompt.md`](prompts/prompt.md) | `ARCHIVED` | Early Phase-1 prompt decomposition that referenced LanceDB before the system standardized on Qdrant (`scientific_papers_gold`) and MongoDB. |
| [`docs/prompts/prompt_backend.md`](prompts/prompt_backend.md) | `ARCHIVED` | Initial B1–B11 step-by-step scaffolding prompt used to bootstrap `backend_rag`. Superseded by [`backend_rag/AGENTS.md`](../backend_rag/AGENTS.md) and [`docs/architecture/ARCHITECTURE_OVERVIEW.md`](architecture/ARCHITECTURE_OVERVIEW.md). |
| [`docs/prompts/prompt_fe_3d.md`](prompts/prompt_fe_3d.md) | `ARCHIVED` | Initial P1–P11 step-by-step bootstrap prompt and verification checklist for `frontend_rag` (`210/210` Vitest, `6/6` Playwright E2E verified). UI design tokens and glass rules are superseded by [`docs/design/FE_DESIGN_SYSTEM.md`](design/FE_DESIGN_SYSTEM.md) and [`docs/prompts/FE3d_prompt_v2.md`](prompts/FE3d_prompt_v2.md). |
| [`docs/research/lesson_2.md`](research/lesson_2.md) | `ARCHIVED` | General course lecture notes on human-AI pair programming methodology (`Requirement → Spec → Pseudocode → Prompt → Verification`). |
