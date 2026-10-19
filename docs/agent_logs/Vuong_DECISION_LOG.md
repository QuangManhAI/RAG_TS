# Decision Log — Scientific Literature RAG (RAG_TS)

Rules: `agents/rules/DECISION_LOG_CONVENTION.md`. Find commits for a decision with
`git log --all --grep="Decision: D-NNN"`.

> **D-001 to D-005 are backfilled** from the exported session `4db53373-1ab6-43ef-a7e9-7bb3fed98130`
> (2026-10-14). They contain only what that export actually shows. Alternatives and
> rationales that were never written down are marked `not recorded`, not guessed.
> Confirmed and accepted by the human on 2026-10-17.

## Index

| ID    | Title                                                                            | Decided by    | Status    |
| ----- | -------------------------------------------------------------------------------- | ------------- | --------- |
| D-001 | Build BE in TypeScript; serve Qwen2.5-7B Q4_K_M locally via node-llama-cpp       | human         | accepted  |
| D-002 | Run all DL models on GPU with CUDA / Metal auto-detection                        | human         | accepted  |
| D-003 | Rerank retrieval as 0.85 × similarity + 0.15 × authority                       | agent-default | accepted  |
| D-004 | Work only on feature branches in FE, BE and root; prefix branches with `vuong` | human         | accepted  |
| D-005 | FE theme: dark and light modes with a switch, built with taste-skill             | human         | accepted  |
| D-006 | Hold PR #1 for multi-session and non-blocking input                              | human         | accepted  |
| D-007 | Query 913k vectors exclusively over Cloudflare R2 S3 storage                     | human         | accepted  |

---

### D-001 — Build BE in TypeScript; serve Qwen2.5-7B Q4_K_M locally via node-llama-cpp

- **Status:** accepted
- **When:** 2026-10-14T15:36:21+07:00
- **Decided by:** human
- **Trigger:** "USE TYPESCRIPTS as this is a TS course" and "use model qwen2.5-7b-instruct-q4_k_m for LLM" (Turn 1)
- **Decision:** The backend is TypeScript. The LLM runs locally through `node-llama-cpp` from `models/qwen2.5-7b-instruct-q4_k_m/`, configured from `.env`.
- **Why:** Course requires TypeScript. Reason for the model choice not recorded.
- **Rejected:** not recorded
- **Touches:** `backend_rag` (LLM module/service), `.env`, `models/`
- **Evidence:** not verified (see D-002 for the live run)

### D-002 — Run all DL models on GPU with CUDA / Metal auto-detection

- **Status:** accepted
- **When:** 2026-10-14T16:09:41+07:00
- **Decided by:** human (layer count is `agent-default`)
- **Trigger:** "run everything DL model on GPU, check for gpu on both mps, apple and window nvidia" (Turn 3)
- **Decision:** `gpu.util.ts` detects NVIDIA CUDA (Windows/Linux) and Apple Silicon Metal (macOS) and falls back to CPU. The LLM offloads 99 layers (agent-chosen value).
- **Why:** Stated need to run on both Windows-NVIDIA and Apple machines.
- **Rejected:** not recorded
- **Touches:** `backend_rag/src/utils/gpu.util.ts`
- **Evidence:** Agent reported one live LoRA query on an RTX 4060 Laptop GPU (CUDA), 14.1 s. Metal path not verified in the log.
- **Revisit if:** 99 layers causes out-of-memory on smaller GPUs.

### D-003 — Rerank retrieval as 0.85 × similarity + 0.15 × authority

- **Status:** accepted
- **When:** 2026-10-14 (reported in the 16:57+07:00 summary)
- **Decided by:** agent-default
- **Trigger:** none found in the log. Check whether `docs/prompt.md` specifies it.
- **Decision:** Search score blends vector similarity (0.85) with co-authorship PageRank authority (0.15).
- **Why:** not recorded
- **Rejected:** not considered
- **Touches:** `backend_rag` (hybrid reranker)
- **Evidence:** not verified. The weights were never benchmarked in the log.
- **Revisit if:** a retrieval-quality benchmark shows different weights perform better.

### D-004 — Work only on feature branches in FE, BE and root; prefix branches with `vuong`

- **Status:** accepted
- **When:** 2026-10-14T17:34:40+07:00 (refined in Turns 12–15)
- **Decided by:** human
- **Trigger:** "i have permission in BE and TS main to push directly but dont want to do that, in FE, i dont have permission therefore branch out" (Turn 11)
- **Decision:** All three repos (FE, BE, root TS) use branches, never direct pushes to `main`. The FE uses a branch inside the FE module, not a fork (Turn 13). Branch names carry the `vuong` prefix and unprefixed ones were deleted (Turns 14–15).
- **Why:** No push permission on FE; direct push on BE/root is permitted but declined by choice.
- **Rejected:** direct push to `main` on BE/root (allowed, declined); FE fork (created by accident, removed)
- **Touches:** git branches in the three repos
- **Evidence:** not verified (final branch names not checked here)

### D-005 — FE theme: dark and light modes with a switch, built with taste-skill

- **Status:** accepted
- **When:** 2026-10-14T18:08:23+07:00
- **Decided by:** human
- **Trigger:** "we want the general theme to be dark orianted and white orianted (can switch theme between those 2 )" (Turn 17); "use the skill to update our FE Leonxlnx/taste-skill" (Turn 18)
- **Decision:** The FE supports dark and light themes with a toggle, styled using the `Leonxlnx/taste-skill`.
- **Why:** not recorded
- **Rejected:** not recorded
- **Touches:** `frontend_rag` (theme/styling)
- **Evidence:** not verified

### D-006 — Hold PR #1 for multi-session and non-blocking input

- **Status:** accepted
- **When:** 2026-10-16T15:47:00+07:00
- **Decided by:** human
- **Trigger:** "Request changes on PR #1 (e.g. support multiple chat threads/sessions, don't disable input during load)"
- **Decision:** PR #1 is held for updates rather than merged; persistence must support multi-session chat threads and non-blocking textarea input, with clear action placed in top navbar.
- **Why:** Single-session overwrites previous scientific discussions; blocking input creates UI lockup.
- **Rejected:** merge PR #1 as-is; reject persistence entirely
- **Touches:** `frontend_rag` (PR #1, ChatPanel, chatStorage)
- **Evidence:** code review and git diff inspection of PR #1 commit `6579a8f`

### D-007 — Query 913k vectors exclusively over Cloudflare R2 S3 storage

- **Status:** accepted
- **When:** 2026-10-16T16:31:18+07:00
- **Decided by:** human
- **Trigger:** "no, we dont want local database, fetch R2 for me, also R2 total embedded text chunks are 900k"
- **Decision:** Uploaded the 913,205-vector consolidated table to Cloudflare R2 and enabled `FORCE_REMOTE_LANCEDB=true` in `.env` to query `s3://uth-scientific-lakehouse/gold/lancedb`.
- **Why:** Centralize vector storage in the R2 Lakehouse cloud rather than relying on local machine disk copies.
- **Rejected:** keep local filesystem LanceDB table as default primary retriever
- **Touches:** `.env`, `backend_rag` (VectorStoreService, inspect-r2.ts)
- **Evidence:** VectorStoreService search via R2 returned 20 chunks against 913,205 vectors in 12,479 ms

### D-008 — Enforce >90% test coverage threshold with zero regressions across frontend and backend

- **Status:** accepted
- **When:** 2026-10-19T22:20:00+07:00
- **Decided by:** human
- **Trigger:** "check the current test coverage and increase it to >90%", "zero test-for-pass regressions", "switch to vite 3d and make test there"
- **Decision:** Switch to `frontend_rag` branch `vuong/vite_3d`, create comprehensive test suites achieving >90% lines/statements/functions coverage (168 tests, 19 files, 96.91% statements, 92.85% functions), and enforce `coverage.thresholds: { lines: 90, statements: 90, functions: 90 }` in `vite.config.ts`. Expand `backend_rag` unit tests to 41 tests (10 test suites) with clean teardown hooks (`onModuleDestroy`) and fast ESM/GPU mocking.
- **Why:** Ensure high production quality, verify all 3D-to-2D screen projection anchors and state stores without flaky Canvas WebGL regressions, and maintain zero test-for-pass regressions.
- **Rejected:** Mocking or deleting existing tests to pass coverage artificially; testing pure WebGL Three.js render loops in Vitest (deferred to Playwright e2e).
- **Touches:** `frontend_rag` (`vite.config.ts`, `src/**/*.test.{ts,tsx}`), `backend_rag` (`src/**/*.spec.ts`, `package.json`), `README.md`
- **Evidence:** Frontend `vitest run --coverage` passes 19/19 files (168/168 tests) with 96.91% statements and 92.85% functions; Backend `jest` passes 10/10 suites (41/41 tests) in 10.9s.

