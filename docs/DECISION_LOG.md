# Decision Log — Scientific Literature RAG (RAG_TS)

Rules: `agents/rules/DECISION_LOG_CONVENTION.md`. Find commits for a decision with
`git log --all --grep="Decision: D-NNN"`.

> **D-001 to D-005 are backfilled** from the exported session `4db53373-1ab6-43ef-a7e9-7bb3fed98130`
> (2026-10-14). They contain only what that export actually shows. Alternatives and
> rationales that were never written down are marked `not recorded`, not guessed.
> All remain `proposed` until confirmed by the human.

## Index

| ID | Title | Decided by | Status |
|---|---|---|---|
| D-001 | Build BE in TypeScript; serve Qwen2.5-7B Q4_K_M locally via node-llama-cpp | human | proposed |
| D-002 | Run all DL models on GPU with CUDA / Metal auto-detection | human | proposed |
| D-003 | Rerank retrieval as 0.85 × similarity + 0.15 × authority | agent-default | proposed |
| D-004 | Work only on feature branches in FE, BE and root; prefix branches with `vuong` | human | proposed |
| D-005 | FE theme: dark and light modes with a switch, built with taste-skill | human | proposed |

---

### D-001 — Build BE in TypeScript; serve Qwen2.5-7B Q4_K_M locally via node-llama-cpp
- **Status:** proposed
- **When:** 2026-10-14T15:36:21+07:00
- **Decided by:** human
- **Trigger:** "USE TYPESCRIPTS as this is a TS course" and "use model qwen2.5-7b-instruct-q4_k_m for LLM" (Turn 1)
- **Decision:** The backend is TypeScript. The LLM runs locally through `node-llama-cpp` from `models/qwen2.5-7b-instruct-q4_k_m/`, configured from `.env`.
- **Why:** Course requires TypeScript. Reason for the model choice not recorded.
- **Rejected:** not recorded
- **Touches:** `backend_rag` (LLM module/service), `.env`, `models/`
- **Evidence:** not verified (see D-002 for the live run)

### D-002 — Run all DL models on GPU with CUDA / Metal auto-detection
- **Status:** proposed
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
- **Status:** proposed
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
- **Status:** proposed
- **When:** 2026-10-14T17:34:40+07:00 (refined in Turns 12–15)
- **Decided by:** human
- **Trigger:** "i have permission in BE and TS main to push directly but dont want to do that, in FE, i dont have permission therefore branch out" (Turn 11)
- **Decision:** All three repos (FE, BE, root TS) use branches, never direct pushes to `main`. The FE uses a branch inside the FE module, not a fork (Turn 13). Branch names carry the `vuong` prefix and unprefixed ones were deleted (Turns 14–15).
- **Why:** No push permission on FE; direct push on BE/root is permitted but declined by choice.
- **Rejected:** direct push to `main` on BE/root (allowed, declined); FE fork (created by accident, removed)
- **Touches:** git branches in the three repos
- **Evidence:** not verified (final branch names not checked here)

### D-005 — FE theme: dark and light modes with a switch, built with taste-skill
- **Status:** proposed
- **When:** 2026-10-14T18:08:23+07:00
- **Decided by:** human
- **Trigger:** "we want the general theme to be dark orianted and white orianted (can switch theme between those 2 )" (Turn 17); "use the skill to update our FE Leonxlnx/taste-skill" (Turn 18)
- **Decision:** The FE supports dark and light themes with a toggle, styled using the `Leonxlnx/taste-skill`.
- **Why:** not recorded
- **Rejected:** not recorded
- **Touches:** `frontend_rag` (theme/styling)
- **Evidence:** not verified
