# DECISION_LOG.md — RAG 3D Knowledge Tree Architectural & Performance Decision Log

- **Motivation/Background**: Track architectural and performance engineering choices across RAG_TS, frontend_rag, and backend_rag to prevent regressions and provide clear provenance.
- **Purpose**: Authoritative record of engineering decisions, trade-offs, and verification results made during performance optimization, SRE stress testing, and debugging sessions.
- **Overview Pipeline**: Git branch isolation → connection & compile fixes → streaming AST layout deferral → 3D draw call batching & frustum clipping → SRE open-workload stress testing & caching optimization.
- **Detailed Plan**: Index table followed by immutable 12-line decision blocks with strict attribution and command-level verification evidence.
- **References**: [agents/template/DECISION_LOG_TEMPLATE.md](../../agents/template/DECISION_LOG_TEMPLATE.md), [frontend_rag/docs/perf-chat-report.md](../../frontend_rag/docs/perf-chat-report.md), [backend_rag/benchmark/BOTTLENECK_ANALYSIS_AND_OPTIMIZATION.md](../../backend_rag/benchmark/BOTTLENECK_ANALYSIS_AND_OPTIMIZATION.md).
- **Created**: 2026-10-20T19:00:00+07:00
- **Last Updated**: 2026-11-06T19:55:00+07:00

[STATUS: ACTIVE]

---

## Index

| ID    | Title                                                                            | Decided by                     | Status   |
| ----- | -------------------------------------------------------------------------------- | ------------------------------ | -------- |
| D-001 | Work exclusively on dedicated performance branch `vuong/perf-benchmark`        | human                          | accepted |
| D-002 | Tunnel remote MongoDB via Cloudflare Access daemon on localhost:27018            | human                          | accepted |
| D-003 | Remove deprecated compiler flag `--ignoreDeprecations 5.0` in backend tsconfig | agent-default                  | accepted |
| D-004 | Auto-detect HTTPS port 443 for Qdrant client connection                          | agent-default                  | accepted |
| D-005 | Defer streaming Markdown rendering to eliminate main-thread layout freezing      | agent-proposed, human-approved | accepted |
| D-006 | Batch Top-K Threads into single LineSegments draw call with vertex colors        | human                          | accepted |
| D-007 | Implement early frustum clipping on 150k canopy points vertex shader             | human                          | accepted |
| D-008 | Enable Three.js frustum culling on all world and effect meshes                   | human                          | accepted |
| D-009 | Maintain single merged mesh for tree trunk and branches                          | human                          | accepted |
| D-010 | Alleviate GPU fill-rate bottleneck via DPR clamp and composer MSAA removal        | agent-proposed, human-approved | accepted |
| D-011 | Optimize render pipeline for 100+ FPS high-refresh rate displays                  | human                          | accepted |
| D-012 | Realtime PerfHUD with mean and standard deviation FPS monitoring                  | human                          | accepted |
| D-013 | Decouple PerfSampler inside Canvas from PerfHUD outside Canvas                   | agent-proposed, human-approved | accepted |
| D-014 | Optimize Web Vitals to achieve LCP < 2.5s and FCP < 450ms                        | agent-proposed, human-approved | accepted |
| D-015 | k6 stress testing and Jupyter Notebook visualization for P95/P99 latency & queue cliff | agent-proposed, human-approved | accepted |
| D-016 | 5-Minute Sustained Qualification Soak Test & Concurrency Step-Down Qualification | human | accepted |
| D-017 | Offline-First Local Chat Persistence via IndexedDB (Dexie.js) with Multi-Turn Sessions | human | accepted |
| D-018 | Production Lighthouse Performance 97 Audit & Minimalist 12-Slide Report Deck | human | accepted |
| D-019 | Enforce Rule L1 3-minute continuous hold-out and P95 < 500ms / P99 < 1000ms SLA for CU certification | human | accepted |
| D-020 | Upgrade authenticated load testing to a 3-stage Production SRE Open-Workload harness | agent-proposed, human-approved | accepted |
| D-021 | Parallelize MongoDB queries, use 1-RTT atomic refresh rotation, and add generation-versioned LRUCache | agent-proposed, human-approved | accepted |
| D-022 | Keep UV_THREADPOOL_SIZE=4 and maxPoolSize=120; revert UV_THREADPOOL_SIZE=12 and maxPoolSize=250 | agent-default | accepted |
| D-023 | Hide the canvas via stylePath and allow a 600 px noise floor in UI screenshots | agent-default | accepted |
| D-024 | Seed a signed-in user in E2E instead of mocking the auth endpoint | agent-default | accepted |
| D-025 | Defer the --ink-faint CSS variable from P16 to P17 | agent-default | accepted |
| D-026 | Rewrite E2E c to assert a cited answer instead of a still-growing one | agent-proposed, human-approved | accepted |
| D-027 | Run the design-system migration on branch `quangmanh/design-system` | agent-default | accepted |
| D-028 | Map off-scale font sizes by role, not by nearest value | agent-proposed, human-approved | accepted |
| D-029 | Size icons by the "text + 2 px" rule over the merge column | agent-proposed, human-approved | accepted |
| D-030 | Keep HistoryDrawer an opaque overlay instead of a Glass surface | agent-default | accepted |
| D-031 | Benchmark on a dedicated 127.0.0.1 port with interleaved A/B runs | agent-default | accepted |
| D-032 | Draw the Liquid Glass specular as a rim light, not a full-surface wash | agent-default | accepted |
| D-033 | Drop mix-blend-mode from the specular layer (breaks backdrop blur) | agent-default | abandoned |
| D-034 | Write the light direction per lit surface via non-inherited @property | agent-default | accepted |
| D-035 | Apply the +2 % level-2 budget to the frame-time median | agent-default | accepted |
| D-047 | Redesign the whole API around the 8 REST rules, replacing routes in v1 | human | accepted |
| D-048 | Use PATCH for partial updates and PUT for full replacement | agent-proposed, human-approved | accepted |
| D-049 | Keep cursor pagination instead of page and size | agent-default | accepted |
| D-050 | Idempotent chat through an Idempotency-Key header and a chat_requests collection | agent-proposed, human-approved | accepted |
| D-051 | Invalidate the whole session list cache on rename | agent-default | accepted |
| D-052 | Negotiate JSON and XML per request, with XML behind a hardened parser | human | accepted |
| D-053 | Add HATEOAS links through a route decorator and an entry point | human | accepted |
| D-054 | Declare proxy hops per deployment instead of trusting one in production | agent-proposed, human-approved | accepted |
| D-055 | Cap running chat requests per client | agent-proposed, human-approved | accepted |
| D-056 | Run the app's MongoDB with authentication on loopback | agent-proposed, human-approved | accepted |
| D-057 | Override vulnerable transitive dependencies | agent-proposed, human-approved | accepted |
| D-058 | Read chunk text from `rag_ts_corpus` | agent-proposed, human-approved | accepted |
| D-059 | Run Laya on Node: fused rerank plus evidence and groundedness gates | human-directed, agent-proposed | accepted |
| D-060 | 5-Scenario Retrieval Ranking & Full 200-Sample DeepEval Generation Benchmark Certification | human-directed, agent-proposed | accepted |

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
- **Revisit if:** TypeScript compiler options are migrated to a shared base tsconfig.

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
- **Revisit if:** Qdrant connection string format is refactored to a single full URL env var.

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
- **Revisit if:** Streaming parser is replaced by an incremental Web Worker Markdown tokenizer.

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
- **Revisit if:** Variable line width per thread is required in future WebGPU renderer.

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
- **Revisit if:** Point count scales beyond 1M and requires octree spatial chunking.

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
- **Revisit if:** Procedural shader displacements exceed static bounding sphere radius.

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
- **Revisit if:** Interior walk-through camera mode is added where 90% of branches are behind the camera.

### D-010 — Alleviate GPU fill-rate bottleneck via DPR clamp and composer MSAA removal

- **Status:** accepted
- **When:** 2026-10-20T21:24:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "it still never get on 60fps... do we have any optimize to do? this web is not at all usable for normal user"
- **Decision:** Clamped canvas `dpr` to `[1, 1.35]`, removed EffectComposer multisampling (`multisampling={0}`), reduced bloom mipmap blur levels to 5, set canvas `antialias: false`, and increased canopy shader `uAlphaCut` to 0.055.
- **Why:** Full tree rendering at DPR 2.0 with 4x MSAA saturated GPU memory bandwidth with 33M sub-samples per frame and dense particle overdraw.
- **Rejected:** Disabling bloom entirely — degraded visual aesthetics of the glowing knowledge tree.
- **Touches:** `WorldCanvas.tsx`, `PostFX.tsx`, `AdaptiveQuality.tsx`, `VectorCanopy.tsx`.
- **Evidence:** `npm run typecheck` passed (0 errors); Vitest 201/201 tests passed.
- **Revisit if:** Dedicated GPU hardware tier detection is added.

### D-011 — Optimize render pipeline for 100+ FPS high-refresh rate displays

- **Status:** accepted
- **When:** 2026-10-20T23:05:00+07:00
- **Decided by:** human
- **Trigger:** "can you increase it to like 100fps? any other optimize?"
- **Decision:** Set DPR to 1.15, disabled stencil buffer (`stencil: false`), reduced bloom resolutionScale to 0.35 with 4 levels, raised canopy `uAlphaCut` to 0.08, and optimized twinkle branch logic.
- **Why:** 100+ FPS targets an 8–10ms frame budget requiring reduced fullscreen blur pass cost and tighter particle alpha discard.
- **Rejected:** Completely removing bloom or fireflies — degraded key visual aesthetic.
- **Touches:** `WorldCanvas.tsx`, `PostFX.tsx`, `AdaptiveQuality.tsx`, `VectorCanopy.tsx`, `canopyShader.ts`.
- **Evidence:** `npx tsc --noEmit` passed with 0 errors; Vitest 201/201 tests passed.
- **Revisit if:** High-DPI 4K monitors require a user-selectable "Ultra Quality" preset.

### D-012 — Realtime PerfHUD with mean and standard deviation FPS monitoring

- **Status:** accepted
- **When:** 2026-10-20T23:40:00+07:00
- **Decided by:** human
- **Trigger:** "update the scripst to monitor it in realtime, or logs, and make perf to display the means and deviation fps"
- **Decision:** Built on-screen `PerfHUD.tsx` with live rolling mean FPS, standard deviation ($\sigma$), min/max, frame time jitter, tier toggles, and console streaming.
- **Why:** Real-time visibility into statistical jitter and hardware bottlenecks during live user interaction and benchmarking.
- **Rejected:** Heavy canvas-based third-party HUD (`r3f-perf`) — caused render overhead and lacked mean/deviation stats.
- **Touches:** `PerfHUD.tsx`, `WorldCanvas.tsx`, `scripts/perf-chat-benchmark.cjs`.
- **Evidence:** `npm run test:perf` logged realtime 1s stream and calculated `57.1 ± 1.2 FPS`, LongTasks = 0.
- **Revisit if:** Telemetry needs to be exported directly to Prometheus/Grafana.

### D-013 — Decouple PerfSampler inside Canvas from PerfHUD outside Canvas

- **Status:** accepted
- **When:** 2026-10-20T23:48:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "R3F: Span is not part of the THREE namespace! Did you forget to extend? at PerfHUD.tsx:156"
- **Decision:** Split `PerfHUD.tsx` into `PerfSampler` (Three.js component running `useFrame`, returning `null`) inside `<Canvas>` and `PerfHUD` (pure HTML DOM overlay) outside `<Canvas>`.
- **Why:** In React Three Fiber (R3F), children inside `<Canvas>` are processed by the custom Three.js reconciler; rendering raw DOM nodes inside `<Canvas>` causes a fatal crash.
- **Rejected:** Wrapping inside `@react-three/drei` `<Html>` component — introduces CSS3D matrix transform calculations and unnecessary DOM synchronization overhead inside the WebGL loop.
- **Touches:** `frontend_rag/src/components/world/overlay/PerfHUD.tsx`, `frontend_rag/src/components/world/WorldCanvas.tsx`.
- **Evidence:** Zero console errors; live HUD rendering with instant/mean/stddev FPS; Vitest 201/201 passed; benchmark passed with 437.3 ± 159.6 FPS.
- **Revisit if:** R3F reconciler architecture changes in a major version upgrade.

### D-014 — Optimize Web Vitals to achieve LCP < 2.5s and FCP < 450ms

- **Status:** accepted
- **When:** 2026-10-21T11:32:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "reduce the LCP to be good <2.5s" (prior audit: FCP 3.6s, LCP 6.6s, Speed Index 4.8s).
- **Decision:** Implemented non-blocking Google Fonts preload, pre-rendered critical loading shell in `index.html`, deferred KaTeX CSS into `Markdown.tsx`, removed artificial `sleep(400)` in `client.ts`, and deferred procedural world building to `requestAnimationFrame`.
- **Why:** Eliminating render-blocking CSS and pre-rendering the HTML splash ensures Google Core Web Vitals "Good" ratings under 4G + 4x CPU throttling.
- **Rejected:** Disabling Google Fonts entirely — degrades typography aesthetics.
- **Touches:** `index.html`, `src/index.css`, `src/components/chat/Markdown.tsx`, `src/api/client.ts`, `src/App.tsx`, `src/components/layout/AppShell.tsx`.
- **Evidence:** Lighthouse 4G + 4x CPU slowdown: FCP 444 ms (-87.7%), LCP 2,152 ms (-67.4%, < 2.5s SLA); Vitest 202/202 and Jest 148/148 passed.
- **Revisit if:** New heavy CSS bundles or synchronous scripts are added to `index.html`.

### D-015 — k6 stress testing and Jupyter Notebook visualization for P95/P99 latency & queue cliff

- **Status:** accepted
- **When:** 2026-10-28T09:05:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "k6 --version, make a stress test, P95, P99, request time, error rate, rapm up user gradually or use jump to identify the max concurent user over acceptable latency" and "make use of notebooks for the test, also, visualize using seaborn or matplotlib"
- **Decision:** Implemented a multi-scenario k6 benchmark suite (`k6_fast_api.js`, `k6_rag_chat.js`) and a Jupyter Notebook (`rag_k6_benchmark_analysis.ipynb`) with Seaborn/Matplotlib statistical charts.
- **Why:** Accurately evaluate throughput, latency distribution (P50, P90, P95, P99), and empirical queue saturation boundaries (`LLM_CONCURRENCY=1`, `LLM_MAX_QUEUE=8`).
- **Rejected:** Running k6 in Docker without persistent logs — native Windows k6 execution with streaming JSON logs allows direct integration with local Python/Jupyter pipelines.
- **Touches:** `backend_rag/benchmark/k6_fast_api.js`, `backend_rag/benchmark/k6_rag_chat.js`, `backend_rag/benchmark/rag_k6_benchmark_analysis.ipynb`, `backend_rag/benchmark/run_analysis.py`, `backend_rag/benchmark/generate_notebook.py`.
- **Evidence:** Fast API served 37,215 requests across 100 VUs (0% failure, P95: 8.63ms, 371.8 req/s); RAG Chat verified queue capacity boundary at 8–9 VUs and immediate HTTP 429 shed at >9 VUs.
- **Revisit if:** Authenticated database-backed endpoints are added to the stress test suite (superseded for auth endpoints by D-019 and D-020).

### D-016 — 5-Minute Sustained Qualification Soak Test & Concurrency Step-Down Qualification

- **Status:** accepted
- **When:** 2026-10-28T11:45:00+07:00
- **Decided by:** human
- **Trigger:** "qualification soak test at candidate concurrency 2000-2500 VUs", "why at queue slot 7,8,9 it clearly exceed 60s timeout but still have green indicate pass?", and "what are the two peak/throughtput p95 shutdown at around 50s and 30s? backend_rag\benchmark\results\qualification_5min_continuous_timeline.png"
- **Decision:** Executed concurrency step-down protocol from 2,500 VUs and 2,000 VUs to qualify **1,600 CU** on stateless Fast API over 300s continuous hold, documented V8 Full GC pauses, and marked burst queue slots 7–9 as `[200* SLA Breach]`.
- **Why:** Prevent misleading SLA compliance interpretations, explain OS socket/V8 runtime phenomena, and produce an auditable benchmark report.
- **Rejected:** Masking or dropping queue slots 7–9 — would obscure queue latency realities under GPU batch constraints.
- **Touches:** `backend_rag/benchmark/generate_burst_empirical_plot.py`, `backend_rag/benchmark/generate_qualification_5min_plots.py`, `docs/benchmark_report.html`.
- **Evidence:** Commits `b2f17b2` (`backend_rag`) and `95b6732` (`RAG_TS`); 1,187,560 requests completed at 1,600 CU (3,958.53 req/s, P95: 379.5ms < 500ms SLA, 0.00% HTTP error rate).
- **Revisit if:** Backend migrates to horizontal multi-worker cluster mode.

### D-017 — Offline-First Local Chat Persistence via IndexedDB (Dexie.js) with Multi-Turn Sessions

- **Status:** accepted
- **When:** 2026-10-28T13:48:00+07:00
- **Decided by:** human
- **Trigger:** "I test out the FE/BE/LLM with the new PR feature, prompt a quesion, wait for it to complte, and reload the page, but i dont see the chat persitance? its clearly a new chat like before... if BE dead for some reason, will FE have persistance chat available for views? that is the main point a web feature that still live if BE die... make the option for 'new chat' too, the name of the chat are the slide of the first prompt content... and beside those chat session, we have the trash bin for chat delete"
- **Decision:** Integrated Dexie.js IndexedDB (`chatDb.ts`) with `useSessionStore` for Cache-First local persistence, auto-titled sessions (`firstQuery.trim().slice(0, 48)`), offline badge (`Ngoại tuyến`), and collapsible `HistoryDrawer.tsx` with 2-step trash delete confirmation.
- **Why:** Direct backend dependency violated the core requirement that past chats must remain persistent, readable, and navigable even when the backend is completely offline.
- **Rejected:** Relying solely on remote REST API `/chat-sessions` (breaks offline) or `localStorage` (5MB quota limit and synchronous serialization blocking).
- **Touches:** `frontend_rag/src/db/chatDb.ts`, `frontend_rag/src/utils/uuid.ts`, `frontend_rag/src/store/sessionStore.ts`, `frontend_rag/src/store/ragStore.ts`, `frontend_rag/src/components/chat/HistoryDrawer.tsx`, `frontend_rag/src/components/chat/ChatPanel.tsx`.
- **Evidence:** `npm run typecheck` exited with code 0; Dexie schema verified with UUIDv7 compatibility; Vitest test suite passed with `fake-indexeddb`.
- **Revisit if:** IndexedDB schema requires version migration for attachment blobs.

### D-018 — Production Lighthouse Performance 97 Audit & Minimalist 12-Slide Report Deck

- **Status:** accepted
- **When:** 2026-10-28T18:15:00+07:00
- **Decided by:** human
- **Trigger:** "make me a report in .html that mimic the minimalistic style of this report preprocessing_slides.html" and "use the newest lighthouse test: 97 Performance, 100 Accessibility, 100 Best Practices, FCP 0.9s, LCP 1.0s, TBT 40ms, CLS 0.002, SI 1.2s"
- **Decision:** Built a self-contained 12-slide presentation in `docs/report_slides.html` with Base64-embedded PNG charts, Unicode NFC normalization, Lighthouse 97 metrics, and 1,600 CU Fast API qualification numbers.
- **Why:** Provide a presentation-ready course defense deliverable without broken external image links or Windows Vietnamese font rendering issues.
- **Rejected:** External image file references — breaks portability when sharing the single HTML file.
- **Touches:** `docs/report_slides.html`, `backend_rag/benchmark/generate_slides.py`, `docs/REPORT.md`.
- **Evidence:** `docs/report_slides.html` generated (2.64 MB), all base64 images rendering, 0 console errors, NFC normalized.
- **Revisit if:** New authenticated SRE stress-test charts need to be appended to the slide deck.

### D-019 — Enforce Rule L1 3-minute continuous hold-out and P95 < 500ms / P99 < 1000ms SLA for CU certification

- **Status:** accepted
- **When:** 2026-10-29T12:15:00+07:00
- **Decided by:** human
- **Trigger:** "1. p95 < 500ms, p99 < 1000ms 2. bump the CUs test to 1000CUs" and "L1: stress test hold out, only comfirm the CUs number if our website can hold for 3 mins"
- **Decision:** Defined strict SLA gates (`P95 < 500ms`, `P99 < 1000ms`, `Error Rate < 1%`) and Rule L1 requiring any candidate Concurrent User (CU) ceiling discovered via ramp-up to pass a continuous 3-minute (`180s`) hold-out before certification.
- **Why:** Short ramp-up windows (20–30s) hide connection pool saturation, cache TTL expiration waves, and V8 garbage collection stalls that appear under sustained load.
- **Rejected:** Certifying peak CUs from a 20-second ramp-up step alone — rejected by human Rule L1 constraint.
- **Touches:** `backend_rag/benchmark/k6_auth_breakpoint.ts`, `backend_rag/benchmark/k6_auth_holdout_3min.ts`, `backend_rag/benchmark/k6_prod_certified_holdout.ts`.
- **Evidence:** `k6 run k6_prod_certified_holdout.ts` validated sustained 180s hold-out with 0.00% errors before certifying capacity.
- **Revisit if:** Production SLA thresholds are tightened to P95 < 250ms after migrating MongoDB off the Cloudflare TCP tunnel.

### D-020 — Upgrade authenticated load testing to a 3-stage Production SRE Open-Workload harness

- **Status:** accepted
- **When:** 2026-10-29T13:30:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "hey argon, what do you think about the way we do stress test? is it the way people usually did? i just want to make sure, everything we did is prodction, what people strive for?"
- **Decision:** Replaced Closed-Model `ramping-vus` and empty-DB read loops with a 3-stage Production SRE suite: Stage A Open-Model `ramping-arrival-rate` (`k6_prod_open_breakpoint.ts`), Stage B 3-minute Hold-Out + 2s server CPU/RAM telemetry (`k6_prod_certified_holdout.ts`, `collect_telemetry.js`), and Stage C `Argon2id` crypto isolation (`k6_prod_auth_crypto.ts`) over 100 users, 1,500 UUIDv7 sessions, and 15,000 messages (`seed_prod_auth_data.js`).
- **Why:** Closed-model `ramping-vus` suffers from Coordinated Omission (throttling request arrivals when server latency rises), and empty MongoDB collections mask index B-tree traversal and serialization costs.
- **Rejected:** Keeping Closed-Model `ramping-vus` with `setup()` login masking and 0 messages in MongoDB — produced inflated, non-production CU numbers.
- **Touches:** `backend_rag/benchmark/seed_prod_auth_data.js`, `backend_rag/benchmark/k6_prod_open_breakpoint.ts`, `backend_rag/benchmark/k6_prod_certified_holdout.ts`, `backend_rag/benchmark/k6_prod_auth_crypto.ts`, `backend_rag/benchmark/collect_telemetry.js`, `backend_rag/benchmark/README.md`.
- **Evidence:** Commit `97bbab6` (`backend_rag`); Stage A–C suite executed end-to-end and established the unoptimized realistic baseline of `100 CUs` (`65 RPS`, `P95 = 259.68ms`, `P99 = 353.35ms`).
- **Revisit if:** Real SSE LLM streaming under concurrent load is added as Stage D on a multi-GPU cluster.

### D-021 — Parallelize MongoDB queries, use 1-RTT atomic refresh rotation, and add generation-versioned LRUCache

- **Status:** accepted
- **When:** 2026-10-29T18:25:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "make a .md documents that analyse the bottle neck of our website and the way to improve it, i want CUs to increase while generally preserve what important by apply smart optimizations"
- **Decision:** Parallelized `ChatSessionsService.get()` via `Promise.all` + `.lean()` + `Singleflight` (`inflightDetail`), added generation-versioned `LRUCache` (`300s` sliding TTL, $O(1)$ `bumpUserGen` invalidation on writes), refactored `TokenService.consume()` to 1-RTT optimistic `findOneAndUpdate` (preserving token family revocation on reuse), cached `login()` lookup/lockout state, aligned compound index `{ session_id: 1, deleted_at: 1, created_at: -1, _id: -1 }`, and enabled HTTP Keep-Alive (`65s`).
- **Why:** Remote MongoDB over `cloudflared.exe` (`localhost:27018`) incurs ~15–25ms TCP tunnel RTT per query; eliminating redundant RTTs and Mongoose hydration boosted certified capacity `4.0x` while preserving all security and data consistency invariants.
- **Rejected:** Lowering `Argon2id` `memoryCost` below `64 MB` or removing refresh token family reuse detection — rejected to preserve OWASP cryptographic security invariants.
- **Touches:** `backend_rag/src/chat-sessions/chat-sessions.service.ts`, `backend_rag/src/chat-sessions/schemas/chat-message.schema.ts`, `backend_rag/src/auth/token.service.ts`, `backend_rag/src/auth/auth.service.ts`, `backend_rag/src/app.module.ts`, `backend_rag/src/main.ts`, `backend_rag/benchmark/BOTTLENECK_ANALYSIS_AND_OPTIMIZATION.md`.
- **Evidence:** Commit `8469e03` (`backend_rag`); `npx jest src/auth/auth.spec.ts src/chat-sessions/chat-sessions.spec.ts` passed `38/38` (`100%`); 3-minute hold-out certified **`400 CUs` (`250 RPS`)** across `48,874` requests (`0.00%` error rate, `P50 = 29.79ms`, `P95 = 265.54ms < 500ms`, `P99 = 475.11ms < 1000ms`).
- **Revisit if:** Backend scales horizontally to multiple Node.js instances, requiring Redis (`ioredis`) to replace in-process `lru-cache`.

### D-022 — Keep UV_THREADPOOL_SIZE=4 and maxPoolSize=120; revert UV_THREADPOOL_SIZE=12 and maxPoolSize=250

- **Status:** accepted
- **When:** 2026-10-29T18:10:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent initiative during D-021 empirical optimization tuning)`
- **Decision:** Pinned `process.env.UV_THREADPOOL_SIZE = '4'` in `main.ts` and `maxPoolSize: 120, minPoolSize: 20` (without `zlib` wire compression) in `app.module.ts`, reverting an experimental trial of `UV_THREADPOOL_SIZE = '12'` and `maxPoolSize: 250, compressors: ['zlib']`.
- **Why:** Running 12 concurrent `64 MB` `Argon2id` hashes (`768 MB` active L3/DRAM churn) caused severe memory-bus thrashing that degraded login `P50` from `2.62s` to `3.32s`, while opening 250 `zlib`-compressed sockets simultaneously stalled `cloudflared.exe` TCP stream multiplexing.
- **Rejected:** Keeping `UV_THREADPOOL_SIZE = '12'` and `maxPoolSize: 250` — empirically slower (`+26.7%` crypto P50 latency) and caused cold-pool socket timeouts at `200 RPS`.
- **Touches:** `backend_rag/src/main.ts`, `backend_rag/src/app.module.ts`, `backend_rag/benchmark/BOTTLENECK_ANALYSIS_AND_OPTIMIZATION.md`.
- **Evidence:** Stage C `k6_prod_auth_crypto.ts` verified `UV_THREADPOOL_SIZE=4` achieved `P50 = 2.62s` (`21.1%` faster than `12` threads) with `0.00%` `GET /auth/me` Event Loop starvation (`P95 = 48.03ms`).
- **Revisit if:** Dedicated bare-metal multi-socket server with 8-channel DDR5 and direct LAN MongoDB replaces the Cloudflare TCP tunnel.

### D-023 — Hide the canvas via stylePath and allow a 600 px noise floor in UI screenshots

- **Status:** accepted
- **When:** 2026-10-30T00:20:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent initiative while executing docs/FE3d_prompt_v2.md P15–P16)`
- **Decision:** `frontend_rag/e2e/visual.spec.ts` hides `<canvas>` and PerfHUD through `stylePath` (`e2e/visual.css`), treats at most 600 differing pixels as "no visual change", and retries a failed shot once.
- **Why:** The canvas covers the viewport, so `mask` painted over the whole UI; identical runs differed by up to 428 glyph pixels (GPU and `--disable-gpu` alike), and roughly 1 run in 9 a 3D-anchored DOM label landed one frame late (13,620 px).
- **Rejected:** `mask: canvas` — hid the entire UI; `--disable-gpu` — same 341–428 px noise and 2× slower; zero-pixel threshold — flaky.
- **Touches:** `frontend_rag/e2e/visual.spec.ts`, `frontend_rag/e2e/visual.css`, `frontend_rag/e2e/visual.spec.ts-snapshots/`.
- **Evidence:** `npx playwright test e2e/visual.spec.ts` on P15 code: 5 consecutive runs 8/8; defining `--ink-faint` produced a 907 px diff on shot 06 and failed as intended.
- **Revisit if:** Baselines must run on another OS/CI (snapshots are `chromium-darwin`) or a real change smaller than 600 px needs to be caught.

### D-024 — Seed a signed-in user in E2E instead of mocking the auth endpoint

- **Status:** accepted
- **When:** 2026-10-29T23:45:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent initiative while executing docs/FE3d_prompt_v2.md P15)`
- **Decision:** `frontend_rag/e2e/helpers.ts` (`seedAuth`) writes a fake user into `localStorage['rag-auth-storage']` before the app boots; no change under `src/`.
- **Why:** Since the Auth panel moved into ChatPanel, mock mode has no `/auth` endpoint, so the "Câu hỏi" input never rendered and E2E c, d, e timed out.
- **Rejected:** Adding a mock auth endpoint in `src/api` — app-code change inside a prompt that forbids touching `src/`.
- **Touches:** `frontend_rag/e2e/helpers.ts`, `frontend_rag/e2e/knowledge-tree.spec.ts`, `frontend_rag/e2e/visual.spec.ts`.
- **Evidence:** Before: `npx playwright test e2e/knowledge-tree.spec.ts` 3 passed, 3 failed (timeout waiting for `getByLabel('Câu hỏi')`); after seeding: d and e passed.
- **Revisit if:** Mock mode gains its own auth flow.

### D-025 — Defer the --ink-faint CSS variable from P16 to P17

- **Status:** accepted
- **When:** 2026-10-30T00:05:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent deviation from the P16 task list in docs/FE3d_prompt_v2.md)`
- **Decision:** `uiInkFaint` exists in `tokens.ts`, but `applyCssVars.ts` and `:root` do not emit `--ink-faint` until P17.
- **Why:** `HistoryDrawer.tsx` already references `var(--ink-faint)`; defining it recolors that text, which breaks P16's "no visual change" requirement.
- **Rejected:** Emitting it in P16 and refreshing the baseline — would mix a visible color fix into the no-visual-change step.
- **Touches:** `frontend_rag/src/theme/tokens.ts`, `frontend_rag/src/theme/applyCssVars.ts`, `frontend_rag/src/index.css`, `frontend_rag/src/theme/tokens.test.ts`.
- **Evidence:** With `--ink-faint` defined, `npx playwright test e2e/visual.spec.ts -g 06` failed (907 px); without it, 8/8 passed in two consecutive runs.
- **Revisit if:** Not applicable — closed by P17.

### D-026 — Rewrite E2E c to assert a cited answer instead of a still-growing one

- **Status:** accepted
- **When:** 2026-10-30T00:40:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "Sửa test cho khớp (Recommended)" (option selected by the human in the review dialog, 2026-10-30)
- **Decision:** E2E c now asserts the answer rendered with at least one citation chip, then the existing "Xong" and pull-back checks; it no longer requires the answer to grow after the ResultCard appears.
- **Why:** The answer was already complete (1067 characters) when the ResultCard became visible, reproduced in two runs; the cause was not investigated.
- **Rejected:** Keep the failing test — leaves the suite red; change the typewriter/streaming behavior — outside the design-system scope.
- **Touches:** `frontend_rag/e2e/knowledge-tree.spec.ts`.
- **Evidence:** `npx playwright test e2e/knowledge-tree.spec.ts` 6 passed (56.1s).
- **Revisit if:** Progressive streaming after the ResultCard becomes a product requirement again.

### D-027 — Run the design-system migration on branch `quangmanh/design-system`

- **Status:** accepted
- **When:** 2026-10-29T23:36:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent initiative when starting docs/FE3d_prompt_v2.md P15)`
- **Decision:** All P15+ commits in `frontend_rag` land on `quangmanh/design-system`, branched from `vuong/perf-benchmark` at `6d54674`; the root repo's submodule pointer is not advanced until that branch is pushed.
- **Why:** The migration rewrites UI styling across ~25 files and must stay revertible per prompt without mixing into the performance branch named in D-001.
- **Rejected:** Committing on `vuong/perf-benchmark` — D-001 dedicates it to performance work.
- **Touches:** `git branches` in `frontend_rag`.
- **Evidence:** `git switch -c quangmanh/design-system`; commits `66edee0`, `88d9afe`, `f675a58`, `6a246ec`, `57224a6`, `0cff6d4` on that branch.
- **Revisit if:** The branch is merged or the team wants one shared integration branch.

### D-028 — Map off-scale font sizes by role, not by nearest value

- **Status:** accepted
- **When:** 2026-10-30T09:00:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "Theo vai trò (Recommended)" (option selected by the human in the review dialog, 2026-10-30)
- **Decision:** `text-[0.65rem]` → `text-micro`, `0.68/0.7rem` → `text-caption`, `0.78/0.8/0.82rem` → `text-xs`; reading text in panels (abstract, chunk text, ResultCard title) is 12 px.
- **Why:** Keeps one size per role (§5.2) instead of putting reading text on the 13 px size reserved for controls.
- **Rejected:** Nearest value (0.8/0.82rem → 13 px `text-control`) — smallest pixel change but mixes roles.
- **Touches:** `frontend_rag/src/components/**` (11 files), `docs/FE_DESIGN_SYSTEM.md` Phụ lục B.4.
- **Evidence:** `npm run check:design` STATS font sizes 19 → 9 (7 + 2 em); `npx playwright test e2e/visual.spec.ts` showed 03 at 2.1 % and 04 at 4.2 % (text reflow), approved by the human.
- **Revisit if:** Abstract or chunk text is reported too small to read.

### D-029 — Size icons by the "text + 2 px" rule over the merge column

- **Status:** accepted
- **When:** 2026-10-30T09:00:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "Quy tắc chữ + 2 (Recommended)" (option selected by the human in the review dialog, 2026-10-30)
- **Decision:** Icons are 12 px beside 10–11 px text, 14 px beside 12–13 px text and inside buttons, 16 px in TopBar and panel-header buttons, 28 px for emblems.
- **Why:** §6.2 stated both a rule and a "merged from" column that contradicted it for 13 px icons next to 12 px text.
- **Rejected:** Following the merge column (13 → 12, 15 → 14 regardless of adjacent text).
- **Touches:** 17 component files under `frontend_rag/src/components/`, `docs/FE_DESIGN_SYSTEM.md` §6.2.
- **Evidence:** Applied in commit `57224a6`; screenshots reviewed with P18.
- **Revisit if:** A dense row needs smaller icons than the rule gives.

### D-030 — Keep HistoryDrawer an opaque overlay instead of a Glass surface

- **Status:** accepted
- **When:** 2026-10-30T10:15:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent deviation from the P19 list in docs/FE3d_prompt_v2.md)`
- **Decision:** HistoryDrawer keeps `.surface-opaque` (strong surface over `--bg`) and only drops its `backdrop-blur-md`; it is not wrapped in `Glass`.
- **Why:** It covers the whole chat column as an overlay; `Glass` would add a border, radius and shadow inside the column, and blur under an opaque fill does nothing.
- **Rejected:** `Glass material="strong" nested` as listed in P19 — changes the drawer's frame for no visual benefit.
- **Touches:** `frontend_rag/src/components/chat/HistoryDrawer.tsx`, `docs/FE_DESIGN_SYSTEM.md` §8.3.
- **Evidence:** `backdrop-filter layers per state` test: history 5 → 3; screenshot 06 unchanged.
- **Revisit if:** The drawer becomes a floating panel instead of a full-column overlay.

### D-031 — Benchmark on a dedicated 127.0.0.1 port with interleaved A/B runs

- **Status:** accepted
- **When:** 2026-10-30T10:35:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent initiative after an invalid measurement)`
- **Decision:** `test:perf` runs target a mock dev server started with `--host 127.0.0.1 --port 519x --strictPort`; comparisons build the previous version in a temporary worktree and alternate runs.
- **Why:** A non-mock dev server already held `127.0.0.1:5174`, so `localhost:5174` sometimes hit it and produced median 4–19 ms with 300–400 long tasks.
- **Rejected:** Killing the other dev server — it belongs to the user; reporting the slow runs — they measured the wrong app.
- **Touches:** `frontend_rag/docs/design-migration-report.md` §2, `frontend_rag/ASSUMPTIONS.md` [P19].
- **Evidence:** Interleaved runs on 5195/5196: P18 median 2.0 ms, P19 median 2.0 / 2.0 / 2.1 ms, p95 3.5–3.6 ms, 4 long tasks each.
- **Revisit if:** The benchmark script starts its own server.

### D-032 — Draw the Liquid Glass specular as a rim light, not a full-surface wash

- **Status:** accepted
- **When:** 2026-10-30T14:10:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent initiative after reviewing the first level-2 screenshot)`
- **Decision:** `.glass::after` covers the border box and is masked to a 1.5 px band along the edge (`mask: … content-box exclude …`); `--lg-specular` is 0.6 at level ≥ 2.
- **Why:** The full-surface radial gradient sketched in FE_DESIGN_SYSTEM §10.4 washed a large bright patch across the wide TopBar; §10.1 describes the specular as a highlight on the glass edge.
- **Rejected:** Full-surface gradient at 0.35 — washed-out TopBar in the first `05-settings-lg2` shot.
- **Touches:** `frontend_rag/src/index.css`, `docs/FE_DESIGN_SYSTEM.md` §10.4.
- **Evidence:** `npx playwright test e2e/visual.spec.ts -g lg2 --update-snapshots`; shots reviewed and accepted by the human with P21.
- **Revisit if:** Level 3/4 adds a lensing band that makes the rim redundant.

### D-033 — Drop mix-blend-mode from the specular layer (breaks backdrop blur)

- **Status:** abandoned
- **When:** 2026-10-30T14:30:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent found the regression while reviewing 05-settings-lg2)`
- **Decision:** `mix-blend-mode: plus-lighter` from the §10.4 sketch is not used; the rim blends normally.
- **Why:** A blended child made the `.glass` element a backdrop root in Chromium, so its `::before` stopped blurring and the legend behind the Settings panel showed through sharply.
- **Touches:** `frontend_rag/src/index.css`, `docs/FE_DESIGN_SYSTEM.md` §10.4.
- **Evidence:** A/B probe on the Settings panel at level 3: with `plus-lighter` the legend text is visible (527 px differ from the version without); without it the area is blurred.
- **Revisit if:** Chromium stops treating blended descendants as backdrop roots.

### D-034 — Write the light direction per lit surface via non-inherited @property

- **Status:** accepted
- **When:** 2026-10-30T14:55:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent initiative after the level-2 budget check failed)`
- **Decision:** `--lg-light-x/y` are registered `inherits: false`, `useLiquidLight` writes them on `.glass[data-shape=panel]` and interactive glass only, `::after` reads them with `inherit`, and lit `::after` layers get `will-change: transform`.
- **Why:** Writing the light on `<html>` every frame cost median +11 % while the pointer moved; scoping the writes alone changed nothing, isolating the rim on its own layer removed the cost.
- **Rejected:** Writing on `<html>` as in the P21 spec — median 2.9–3.0 ms vs 2.6 ms; per-surface writes without `will-change` — 2.9–3.0 ms.
- **Touches:** `frontend_rag/src/theme/useLiquidLight.ts`, `frontend_rag/src/index.css`, `frontend_rag/src/theme/useLiquidLight.test.tsx`.
- **Evidence:** Pointer-motion probe (150k vectors, 880 rAF intervals, 3 interleaved runs): level 1 median 2.6–2.7 ms / p95 3.7 ms; level 3 final median 2.5 ms / p95 4.4–5.1 ms; pointer still: 2.6 ms both.
- **Revisit if:** The number of lit surfaces grows enough for the per-frame `querySelectorAll` to show up.

### D-035 — Apply the +2 % level-2 budget to the frame-time median

- **Status:** accepted
- **When:** 2026-10-30T15:05:00+07:00
- **Decided by:** agent-default
- **Trigger:** `none (agent reading of Phụ lục A #2, which states the budget on the frame median)`
- **Decision:** Level 2 passes on median (2.5 vs 2.6 ms with the pointer moving); the higher p95 (4.4–5.1 vs 3.7 ms) is recorded, not used as a gate.
- **Why:** Phụ lục A #2 defines the budget as "frame median so với Cấp 1"; p95 rises only while the pointer moves and stays far below a 60 Hz frame.
- **Rejected:** Blocking level 2 on p95 — not the approved metric.
- **Touches:** `frontend_rag/docs/design-migration-report.md`, `frontend_rag/ASSUMPTIONS.md` [P21].
- **Evidence:** Same probe as D-034; standard `test:perf` at level 3: median 2.2–2.3 ms, p95 3.4 ms, 4 long tasks.
- **Revisit if:** Users report jank while moving the pointer, or the team wants a p95 budget.

### D-047 — Redesign the whole API around the 8 REST rules, replacing routes in v1

- **Status:** accepted
- **When:** 2026-11-04T22:27:54+07:00
- **Decided by:** human
- **Trigger:** "tôi cần bản sửa lại BE và FE - có 1 API Design chuẩn đầy đủ 8 nguyên tắc gì đó, cho toàn bộ các loại API nhé"
- **Decision:** All routes follow `docs/API_DESIGN.md`; old routes are replaced in `/api/v1` with no aliases, and a contract test enforces the route list and naming.
- **Why:** The frontend is the only client and ships with the backend (agent-default detail: no `/api/v2`, no aliases).
- **Rejected:** `/api/v2` next to v1 — two surfaces to maintain with one client.
- **Touches:** `backend_rag/src`, `backend_rag/test/flow.e2e-spec.ts`, `backend_rag/benchmark/k6_*.ts`, `frontend_rag/src/api`, `frontend_rag/src/store`, `docs/API_DESIGN.md`.
- **Evidence:** `npx jest --runInBand`: 210 passed; `npx jest --config test/jest-e2e.json --runInBand`: 13 passed; FE and BE not yet run together at acceptance.
- **Revisit if:** A second API client appears: breaking changes then need `/api/v2`.

### D-048 — Use PATCH for partial updates and PUT for full replacement

- **Status:** accepted
- **When:** 2026-11-04T22:47:55+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "oh no! bạn đừng tin cái hình quá" and "cái ảnh nó lỗi lắm ! nên là cứ tin mình"
- **Decision:** `PATCH /users/me` and `PATCH /chat-sessions/:id`; `PUT` for `/users/me/password` and `/admin/users/:id/roles` and `/status`, whose body is the whole state.
- **Why:** The image's "avoid PATCH" is not part of the human's 8 standards; PATCH leaves room for new optional fields.
- **Rejected:** PUT everywhere — done first to follow the image, then reversed in the same task.
- **Touches:** `backend_rag/src/auth/users.controller.ts`, `backend_rag/src/chat-sessions/chat-sessions.controller.ts`, `frontend_rag/src/store/sessionStore.ts`, `backend_rag/benchmark/k6_*.ts`.
- **Evidence:** Contract test `lists exactly the documented routes` in `backend_rag/test/flow.e2e-spec.ts`.
- **Revisit if:** A resource needs PATCH semantics other than a partial update of its own fields.

### D-049 — Keep cursor pagination instead of page and size

- **Status:** accepted
- **When:** 2026-11-04T22:27:54+07:00
- **Decided by:** agent-default
- **Trigger:** "cái ảnh nó lỗi lắm ! nên là cứ tin mình"; standard 5 of the human's list shows `page`/`size` as an example.
- **Decision:** Every list uses `limit` + keyset `cursor`; the cursor is bound to its `sort`, and `sort` and operator filters are whitelisted per route.
- **Why:** Session lists change constantly, so offsets repeat or skip rows and slow down on deep pages.
- **Rejected:** `page`/`size` — the drawbacks above.
- **Touches:** `backend_rag/src/common/pagination.ts`.
- **Evidence:** `backend_rag/src/common/pagination.spec.ts`, admin cursor test in `backend_rag/src/admin/admin.spec.ts`.
- **Revisit if:** A UI needs to jump to an arbitrary page number.

### D-050 — Idempotent chat through an Idempotency-Key header and a chat_requests collection

- **Status:** accepted
- **When:** 2026-11-04T22:27:54+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "bạn cũng hiểu rồi" (reply to the proposal: replay when finished, 409 while running); header instead of a body field is agent-default.
- **Decision:** With a JWT, a repeated key replays the stored answer, a running or mismatched one gets 409, a failed or aborted one is released, and a crashed one is taken over after 5 minutes.
- **Why:** A retry otherwise reruns the LLM and stores duplicate messages or a duplicate session; `completed` shares the transaction of the messages.
- **Rejected:** `client_request_id` in the body — the header is the common convention; 409 for every repeat — the client would have to reload the session itself.
- **Touches:** `backend_rag/src/chat-sessions/schemas/chat-request.schema.ts`, `backend_rag/src/chat-sessions/chat-sessions.service.ts`, `backend_rag/src/rag/rag.service.ts`.
- **Evidence:** e2e test `a repeated chat request replays the stored answer`, `Idempotency-Key store` tests in `backend_rag/src/chat-sessions/chat-sessions.spec.ts`.
- **Revisit if:** On a standalone MongoDB (no transactions), a crash between the writes could still duplicate an exchange after the lock expires.

### D-051 — Invalidate the whole session list cache on rename

- **Status:** accepted
- **When:** 2026-11-04T22:27:54+07:00
- **Decided by:** agent-default
- **Trigger:** `none (found while adding limit, sort and filter to the cache key)`
- **Decision:** `rename()` bumps the user's generation instead of patching the cached first page.
- **Why:** Patching left filtered and later pages stale for up to 5 minutes, which breaks invariant 4; more cache-key dimensions would make it worse.
- **Rejected:** Keeping the in-place patch for its cache-hit rate — stale data is the worse failure.
- **Touches:** `backend_rag/src/chat-sessions/chat-sessions.service.ts`.
- **Evidence:** not verified under load: the 400 CU certification was not re-run.
- **Revisit if:** A new k6 run shows the extra list misses after rename matter.

### D-052 — Negotiate JSON and XML per request, with XML behind a hardened parser

- **Status:** accepted
- **When:** 2026-11-04T22:47:55+07:00
- **Decided by:** human
- **Trigger:** "Đảm bảo API hỗ trợ nhiều định dạng dữ liệu, chẳng hạn như JSON và XML"
- **Decision:** `Content-Type` and `Accept` choose the format on every route except the SSE stream; no document types are accepted and XML values stay strings except known boolean and list fields (agent-default details, `fast-xml-parser`).
- **Why:** Standard 4 of the human's list; string values keep numeric passwords and titles intact.
- **Rejected:** `parseTagValue` type guessing — a numeric password would fail validation.
- **Touches:** `backend_rag/src/common/xml.ts`, `backend_rag/src/app.setup.ts`, `backend_rag/package.json`.
- **Evidence:** e2e tests `accepts and returns XML` and `rejects malformed XML` in `backend_rag/test/flow.e2e-spec.ts`.
- **Revisit if:** A new DTO adds a boolean or list field: it must be added to the XML field sets.

### D-053 — Add HATEOAS links through a route decorator and an entry point

- **Status:** accepted
- **When:** 2026-11-04T22:47:55+07:00
- **Decided by:** human
- **Trigger:** "Sử dụng Hypermedia as the Engine of Application State (HATEOAS)"
- **Decision:** `@WithLinks` plus a global interceptor add `_links` on copies of the body; `GET /api/v1` is the entry point and action links can depend on state (agent-default details: link shape `href` + `method`).
- **Why:** Copies keep cached bodies clean, and services stay unaware of URLs.
- **Rejected:** Building links inside services — they would then need to know the API prefix.
- **Touches:** `backend_rag/src/common/hateoas.ts`, `backend_rag/src/api-root/api-root.controller.ts`, all controllers.
- **Evidence:** e2e test `responses carry hypermedia links a client can follow`.
- **Revisit if:** The frontend starts following links instead of hard-coded paths.

### D-054 — Declare proxy hops per deployment instead of trusting one in production

- **Status:** accepted
- **When:** 2026-11-05T01:48:40+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "fix lại toàn bộ tất cả các lỗi đó nhé"
- **Decision:** `TRUST_PROXY_HOPS` (default 0) sets `trust proxy`; `docker-compose.yml` publishes the app on loopback by default.
- **Why:** With `trust proxy = 1` and no proxy, `X-Forwarded-For` chooses the client IP, which defeats the rate limiter and login lockout and forges audit IPs.
- **Rejected:** Keeping 1 in production and documenting that a proxy is required — a wrong deployment would fail silently and unsafely.
- **Touches:** `backend_rag/src/common/security.ts`, `backend_rag/src/config/env.schema.ts`, `backend_rag/docker-compose.yml`.
- **Evidence:** `backend_rag/src/common/security.spec.ts`.
- **Revisit if:** A deployment puts a proxy in front of the app: set the variable to its hop count, or all clients share one bucket.

### D-055 — Cap running chat requests per client

- **Status:** accepted
- **When:** 2026-11-05T01:48:40+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "fix lại toàn bộ tất cả các lỗi đó nhé"
- **Decision:** At most 2 running chat requests per user and 3 per IP, in-process; anonymous chat gets 10 requests/min, callers with a token 20 (agent-default values).
- **Why:** One slot plus 8 waiting meant a single anonymous IP within its rate limit could answer every other client with 429 `llm queue is full`.
- **Rejected:** Requiring login for chat — it removes the anonymous mode the product offers.
- **Touches:** `backend_rag/src/rag/client-slots.ts`, `backend_rag/src/rag/rag.controller.ts`, `backend_rag/src/common/throttler.ts`.
- **Evidence:** `backend_rag/src/rag/client-slots.spec.ts`, `concurrent requests per client` in `backend_rag/src/rag/rag.spec.ts`.
- **Revisit if:** k6 chat scenarios from one IP need these limits lifted, or the LLM moves to more than one instance.

### D-056 — Run the app's MongoDB with authentication on loopback

- **Status:** accepted
- **When:** 2026-11-05T01:48:40+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "fix lại toàn bộ tất cả các lỗi đó nhé"
- **Decision:** `--keyFile` (which implies auth) on the single-node replica set, published on `127.0.0.1`; the app uses `ragts_app` with `readWrite` on `rag_ts`, and root stays for administration.
- **Why:** The container accepted anonymous connections on all interfaces, so anyone on the network could read the data or set `roles: ['admin']`.
- **Rejected:** Binding to loopback only, without auth — any local process or later port change would still get full access.
- **Touches:** `backend_rag/docker-compose.yml`, `backend_rag/docker/mongo-init-app-user.js`, `backend_rag/.env.example`.
- **Evidence:** Throwaway compose project: anonymous access denied, app user can run transactions and cannot write other databases. Live: same document counts before and after.
- **Revisit if:** The app must also read `rag_ts_corpus` from this instance: grant `read` on it.

### D-057 — Override vulnerable transitive dependencies

- **Status:** accepted
- **When:** 2026-11-05T01:48:40+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** "fix lại toàn bộ tất cả các lỗi đó nhé"
- **Decision:** npm `overrides` for `simple-git` ^4.0.2, `multer` ^2.5.0 and `undici` ^7.30.1.
- **Why:** Their parents pin vulnerable versions, and `npm audit` only offered downgrades such as `node-llama-cpp@2.0.0`.
- **Rejected:** `npm audit fix --force` — it downgrades `node-llama-cpp` by a major version.
- **Touches:** `backend_rag/package.json`, `backend_rag/package-lock.json`.
- **Evidence:** `npm audit --omit=dev` reports 0; backend tests pass. `simple-git` 4 is a major jump for `node-llama-cpp`: only its build-from-source path uses it, and that path was not exercised.
- **Revisit if:** A parent package updates its range, or `node-llama-cpp` has to build llama.cpp from source.

### D-058 — Read chunk text from `rag_ts_corpus`

- **Status:** accepted
- **When:** 2026-11-05T03:06:00+07:00
- **Decided by:** agent-proposed, human-approved
- **Trigger:** Qdrant no longer stores `text`, so `toChunkPayload` dropped every point and every chat answered "no context".
- **Decision:** `CorpusService` reads `paper_chunks` and `papers` by id in one batched read per page, on its own connection from `CORPUS_MONGO_URI`.
- **Why:** MongoDB is the source of truth for content. A point that still has `text` keeps working.
- **Rejected:** Using the `MONGO_ADMIN_*` account as the app's URI — it has `root`, which D-056 forbids. Startup logs `CORPUS_OVERPRIVILEGED` for such an account.
- **Touches:** `backend_rag/src/corpus/`, `backend_rag/src/vector-store/`, `backend_rag/src/config/env.schema.ts`, `backend_rag/.env.example`.
- **Evidence:** Live chat on port 3100 returned text and citations. The account used for that test was `root`, passed through the process environment only.
- **Revisit if:** A read-only corpus account exists, or the remaining 827 chunks and 11,928+ papers are synced: unsynced chunks are skipped and unsynced papers have no title or authors.

### D-059 — Run Laya on Node: fused rerank plus evidence and groundedness gates

- **Status:** accepted
- **When:** 2026-11-05T03:06:00+07:00
- **Decided by:** human-directed (Laya, TypeScript only, `receptron/laya-onnx`), agent-proposed (defaults)
- **Trigger:** "Laya là 1 giải pháp thay thế cho JEV nên bạn có thể thực hiện ngay bây giờ xem sao" and "đây là môn TS/JS vậy chúng ta sẽ run bằng TS/JS".
- **Decision:** `JudgeService` runs `@receptron/laya` in-process. A context with low relevance gets one LLM query rewrite, then an abstention; an answer with low support is replaced through the `answer_replaced` SSE event. `LAYA_RERANK` defaults to true and fuses the existing order with Laya's through reciprocal rank fusion; false keeps the existing order. `LAYA_EVIDENCE_THRESHOLD` defaults to 0.6, `LAYA_GROUNDED_THRESHOLD` to 0.3. Unset `LAYA_MODEL_DIR` turns all of it off, and any judge failure falls back to the plain pipeline.
- **Why:** On 28 known-answer questions, top 10: vector hit@1 11/28, MRR 0.494; Laya alone 9/28, 0.457; fused 13/28, 0.577. Changing the prompt or using a 0–3 score did not beat vector alone.
- **Rejected:** Laya alone as the ranker, per the numbers above. A Python sidecar, removed at the user's request. Rewriting the ONNX graph by hand, since a published bundle exists.
- **Touches:** `backend_rag/src/judge/`, `backend_rag/src/rag/`, `backend_rag/benchmark/eval-laya.mjs`, `backend_rag/benchmark/eval-laya-variants.mjs`, `backend_rag/benchmark/laya-eval-set.json`, `frontend_rag/src/store/pipeline.ts`, `frontend_rag/src/store/ragStore.ts`, `frontend_rag/src/types/rag.ts`.
- **Evidence:** Top 5 at t=0.6 keeps 25/28 answerable questions; top 10 at t=0.6 rejects 11/14 unanswerable ones. Groundedness at t=0.3 keeps 10/10 supported answers and rejects 3/10 fabricated ones. The set is small, written by the agent, and the thresholds were picked on the same data with no holdout. About 427 ms per Laya call under memory pressure.
- **Revisit if:** A larger labelled set exists, BM25 or hybrid retrieval is added, or the 20 s rewrite on a rejected question proves too slow.

### D-060 — 5-Scenario Retrieval Ranking & Full 200-Sample DeepEval Generation Benchmark Certification

- **Status:** accepted
- **When:** 2026-11-06T18:40:00+07:00
- **Decided by:** human-directed, agent-proposed
- **Trigger:** User requested executing the complete DeepEval generation benchmark across all 200 test cases at $0.00 cost and clarifying the boundary between the 5-scenario retrieval ranking benchmark and generation quality.
- **Decision:** Keep the 5-Scenario Retrieval Ranking Benchmark (Vector, Laya, RRF, PageRank, Full System) at N=120 curated queries with pre-indexed ground-truth passage IDs, while certifying the DeepEval Generation Benchmark across all 200 samples (`rag_goldens_200.json`) on the Full System (`Vector + Laya + PageRank`) using Local Qwen 2.5-7B Instruct on CUDA port 9001 ($0.00 spent).
- **Why:** Ranking metrics (MRR, Hit@k) require exact pre-indexed passage IDs in Qdrant, whereas DeepEval generation evaluates end-to-end LLM synthesis and citation grounding across the full 200-question dataset (Faithfulness 0.775 / 69.5% pass, Academic Citation Grounding 0.632 / 74.1% pass).
- **Rejected:** Running retrieval ranking on unindexed broad questions (which artificially flattened scores to identical 0.3000 MRR), or removing Python files without preserving self-contained HTML/PNG benchmark reports in `RAG_TS`.
- **Touches:** `docs/research/laya-jev.md`, `docs/INDEX.md`, `AGENTS.md`, `backend_rag/benchmark/README.md`.
- **Evidence:** 200/200 test cases certified in `logs/benchmarks/benchmark_report_k5_local_20261106_175925.md` ($0.00 API cost); 5-scenario ranking certified in `logs/benchmarks/batch_101_120_5_scenarios.json`; 0 `.py` files remaining in `RAG_TS`.
- **Revisit if:** Additional ground-truth chunk IDs are annotated and indexed in Qdrant for questions 121–200.
