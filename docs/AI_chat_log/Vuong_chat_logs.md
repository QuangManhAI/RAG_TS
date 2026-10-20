# DECISION_LOG.md — RAG 3D Knowledge Tree Architectural & Performance Decision Log

- **Motivation/Background**: Track architectural and performance engineering choices across RAG_TS, frontend_rag, and backend_rag to prevent regressions and provide clear provenance.
- **Purpose**: Authoritative record of engineering decisions, trade-offs, and verification results made during performance optimization and debugging sessions.
- **Overview Pipeline**: Git branch isolation → connection & compile fixes → streaming AST layout deferral → 3D draw call batching & frustum clipping.
- **Detailed Plan**: Index table followed by immutable 12-line decision blocks with strict attribution and command-level verification evidence.
- **References**: [agents/template/DECISION_LOG_TEMPLATE.md](../../agents/template/DECISION_LOG_TEMPLATE.md), [frontend_rag/docs/perf-chat-report.md](../../frontend_rag/docs/perf-chat-report.md).
- **Created**: 2026-10-20T19:00:00+07:00
- **Last Updated**: 2026-10-20T21:20:00+07:00

[STATUS: ACTIVE]

---

## Index

| ID | Title | Decided by | Status |
|---|---|---|---|
| D-001 | Work exclusively on dedicated performance branch `vuong/perf-benchmark` | human | accepted |
| D-002 | Tunnel remote MongoDB via Cloudflare Access daemon on localhost:27018 | human | accepted |
| D-003 | Remove deprecated compiler flag `--ignoreDeprecations 5.0` in backend tsconfig | agent-default | accepted |
| D-004 | Auto-detect HTTPS port 443 for Qdrant client connection | agent-default | accepted |
| D-005 | Defer streaming Markdown rendering to eliminate main-thread layout freezing | agent-proposed, human-approved | accepted |
| D-006 | Batch Top-K Threads into single LineSegments draw call with vertex colors | human | accepted |
| D-007 | Implement early frustum clipping on 150k canopy points vertex shader | human | accepted |
| D-008 | Enable Three.js frustum culling on all world and effect meshes | human | accepted |
| D-009 | Maintain single merged mesh for tree trunk and branches | human | accepted |

---

### D-001 — Work exclusively on dedicated performance branch `vuong/perf-benchmark`
- **Status:** accepted
- **When:** 2026-10-20T19:40:00+07:00
- **Decided by:** human
- **Trigger:** "do not make changes directly on main, but instead do it on a branch"
- **Decision:** All performance profiling, benchmarking, and optimizations are isolated on git branch `vuong/perf-benchmark`.
- **Why:** Protects `main` stability while running experimental graphics optimizations and benchmarking scripts.
- **Rejected:** Committing directly to `main` — explicitly prohibited by human safety constraint.
- **Touches:** `git branches` across `RAG_TS`, `frontend_rag`, and `backend_rag`.
- **Evidence:** `git status` confirmed all three repositories on `vuong/perf-benchmark`.
- **Revisit if:** Changes are verified, bench-tested, and ready for merge pull request.

### D-002 — Tunnel remote MongoDB via Cloudflare Access daemon on localhost:27018
- **Status:** accepted
- **When:** 2026-10-20T19:20:00+07:00
- **Decided by:** human
- **Trigger:** "Bạn của bạn đang kết nối thẳng tới mongodb.minio-quangmanh.io.vn:27017, và cách đó không chạy được với Cloudflare Tunnel... Muốn đi qua tunnel thì phía bạn ấy cũng phải chạy cloudflared"
- **Decision:** Run local `cloudflared access tcp` pointing `mongodb.minio-quangmanh.io.vn` to local port 27018 and connect backend via `localhost:27018`.
- **Why:** Cloudflare edge terminates direct raw TCP traffic on external 27017; only tunnel client bridging passes the stream.
- **Rejected:** Direct connection to `mongodb.minio-quangmanh.io.vn:27017` — timed out after 30s.
- **Touches:** `backend_rag/.env`, `cloudflared` background daemon process.
- **Evidence:** `curl http://localhost:3000/api/v1/health` returned `"mongo": true`.
- **Revisit if:** MongoDB is migrated to a direct private subnet or cloud-native Atlas instance.

### D-003 — Remove deprecated compiler flag `--ignoreDeprecations 5.0` in backend tsconfig
- **Status:** accepted
- **When:** 2026-10-20T19:10:00+07:00
- **Decided by:** agent-default
- **Trigger:** "error TS5103: Invalid value for '--ignoreDeprecations'"
- **Decision:** Removed `--ignoreDeprecations 5.0` compiler flag from `backend_rag/tsconfig.json`.
- **Why:** TypeScript 5.8+ deprecated and removed acceptance of value `5.0` for this flag, breaking watch compilation.
- **Rejected:** Downgrading project TypeScript version — would cause dependency drift and node_modules churn.
- **Touches:** `backend_rag/tsconfig.json`.
- **Evidence:** `npm run start:dev` compiled with 0 errors.

### D-004 — Auto-detect HTTPS port 443 for Qdrant client connection
- **Status:** accepted
- **When:** 2026-10-20T19:25:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent initiative)`
- **Decision:** Configured `QdrantClient` in `vector-store.module.ts` to automatically detect port 443 as `https: true`.
- **Why:** Qdrant client defaulted to HTTP when port 443 was passed without an explicit TLS flag, causing connection reset.
- **Rejected:** Hardcoding `https` scheme in host string — would break local development HTTP Qdrant containers.
- **Touches:** `backend_rag/src/vector-store/vector-store.module.ts`.
- **Evidence:** `curl http://localhost:3000/api/v1/health` returned `"qdrant": true`.

### D-005 — Defer streaming Markdown rendering to eliminate main-thread layout freezing
- **Status:** accepted
- **When:** 2026-10-20T20:10:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "it some time have lag spike especially when the animation ply out with the chat, what kind of measurement that we have been implement to mitigate that"
- **Decision:** Implemented `useDeferredValue` and micro-batched 30ms token streaming buffer in `ragStore.ts` and `Markdown.tsx`.
- **Why:** KaTeX and remark-math AST parsing on every individual streamed token blocked the browser main thread.
- **Rejected:** Raw unformatted text streaming — poor user experience lacking math typography.
- **Touches:** `frontend_rag/src/components/chat/Markdown.tsx`, `frontend_rag/src/store/ragStore.ts`.
- **Evidence:** Vitest 201/201 tests passed; frame median latency reduced to 0.8ms.

### D-006 — Batch Top-K Threads into single LineSegments draw call with vertex colors
- **Status:** accepted
- **When:** 2026-10-20T20:50:00+07:00
- **Decided by:** human
- **Trigger:** "we need more optimization, i dont nkow much about web 3d, bu t3d game usually do some otherkind of optimization, group and reduce draw call"
- **Decision:** Replaced per-thread `<Line>` components in `TopKThreads.tsx` with a single `THREE.LineSegments` mesh using vertex colors.
- **Why:** Reduced $K$ separate WebGL draw calls down to 1 single batched draw call.
- **Rejected:** Instanced fat lines — higher vertex shader complexity with no visual difference.
- **Touches:** `frontend_rag/src/components/world/fx/TopKThreads.tsx`.
- **Evidence:** Vitest 201/201 tests passed; WebGL draw calls reduced during query flight.

### D-007 — Implement early frustum clipping on 150k canopy points vertex shader
- **Status:** accepted
- **When:** 2026-10-20T20:51:00+07:00
- **Decided by:** human
- **Trigger:** "dont draw anything that are notl visiible to the camera/user tries that"
- **Decision:** Added near-plane clipping ($mv.z \ge -0.1$) and guard-band NDC clipping ($|clip.x|, |clip.y| > 1.25w$) at start of `CANOPY_VERT`.
- **Why:** Prunes up to 150,000 points before rasterization when camera looks away or focuses on a sub-branch.
- **Rejected:** CPU-side point filtering every frame — CPU array allocations for 150k points caused 50ms stutters.
- **Touches:** `frontend_rag/src/components/world/canopy/canopyShader.ts`.
- **Evidence:** `npm test` passed 201/201 tests; benchmark frame time median reached 0.8ms (~1250 FPS).

### D-008 — Enable Three.js frustum culling on all world and effect meshes
- **Status:** accepted
- **When:** 2026-10-20T20:52:00+07:00
- **Decided by:** human
- **Trigger:** "dont draw anything that are notl visiible to the camera/user tries that"
- **Decision:** Computed valid bounding spheres and enabled `frustumCulled={true}` across KnowledgeBlocks, SapFlow, Fireflies, EssenceFlow, QueryText, and streamParticles.
- **Why:** Meshes previously had `frustumCulled={false}`, forcing GPU rendering even when completely off-screen.
- **Rejected:** Leaving `frustumCulled={false}` — wasted GPU cycles on off-screen terrain and particles.
- **Touches:** `KnowledgeBlocks.tsx`, `SapFlow.tsx`, `Fireflies.tsx`, `EssenceFlow.tsx`, `QueryText.tsx`, `CompareThreads.tsx`, `streamParticles.ts`.
- **Evidence:** Full tree renders 140k triangles; camera zoom-in drops triangle count to 87k (53k terrain triangles culled).

### D-009 — Maintain single merged mesh for tree trunk and branches
- **Status:** accepted
- **When:** 2026-10-20T21:18:00+07:00
- **Decided by:** human
- **Trigger:** "i think we stop remove hidden geomtry as the structure of the web means that we see the whole tree almost all the times"
- **Decision:** Preserved `AncientTree` / `Branches.tsx` as a single merged mesh with 1 draw call rather than splitting into multiple branch clusters.
- **Why:** The application UI keeps the tree in view almost permanently, making mesh clustering add draw call overhead with zero culling benefit.
- **Rejected:** Hierarchical branch chunking into 8-10 draw calls — rejected by human constraint.
- **Touches:** `frontend_rag/src/components/world/tree/Branches.tsx`.
- **Evidence:** Draw calls remain at absolute minimum (1 draw call for entire tree).
