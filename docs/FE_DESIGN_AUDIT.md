# Rà soát Frontend — Quy trình prompt, Quy chuẩn thiết kế & Liquid Glass

- **Ngày rà soát:** 2026-10-29
- **Phạm vi:** submodule `frontend_rag` tại commit `d035b48` (2026-10-28, `fix(sync): enable bidirectional chat sync…`)
- **Câu hỏi cần trả lời:**
  1. FE hiện tại đang như thế nào?
  2. Quy trình prompt đã diễn ra ra sao, hiện ở tình trạng nào?
  3. Đã có quy chuẩn thiết kế chung (màu, bo góc, độ mờ…) chưa?
  4. Đã có Liquid Glass chưa?

## Tóm tắt

Phần 3D, logic và test thì vững. Lớp UI (DOM) thì **không có quy chuẩn thiết kế**: chỉ màu có token, còn bo góc, độ mờ, độ trong, đổ bóng, cỡ chữ và khoảng cách thì không có. Mỗi lần prompt lại tự đặt ra một bộ số mới, nên giao diện ngày càng lệch nhau. Nhận xét của thầy ("soát FE như vậy thì dự án sẽ nổ tung") khớp với số liệu đo được dưới đây.

**Liquid Glass thật chưa có.** Thứ đang mang tên `liquid-*` chỉ là glassmorphism thông thường, và chỉ dùng ở form đăng nhập.

---

## 1. Hiện trạng FE

**Stack:** Vite 6 + React 19 + TypeScript, `@react-three/fiber` / `drei` / `postprocessing` (three.js 0.186), Zustand, Tailwind CSS v4, Dexie (IndexedDB), react-markdown + KaTeX, Vitest + Playwright.

**Cấu trúc `src/`:**

| Thư mục | Nội dung |
| :-- | :-- |
| `components/world/` | Cảnh 3D: cây (rễ, thân, cành, nhựa sống), tán lá vector, mặt đất/khối tri thức, hiệu ứng (fx), overlay, camera, PostFX |
| `components/chat/` | ChatPanel, MessageList/Bubble, Markdown, CitationChip, PipelineStatus, HistoryDrawer |
| `components/layout/` | AppShell, TopBar, SettingsPanel, CrawlSettings, CategoryLegend, YearSlider, HealthBanner, WorldError |
| `components/result/` | ResultCard, PaperPanel, ChunkList, ScoreBars, TopKBadges |
| `components/auth/` | AuthOverlay (form "Liquid Glass") |
| `store/` | Zustand: rag, ui, world, fx, crawl, auth, session, pipeline |
| `theme/` | `tokens.ts`, `applyCssVars.ts`, `useTheme.ts` |
| `db/` | `chatDb.ts` (Dexie, chat offline) |
| `index.css` | 862 dòng CSS toàn cục (glass, btn, liquid-*, animation…) |

**Đã làm được:** toàn bộ thế giới 3D, luồng hỏi đáp có animation 6 giai đoạn, thẻ kết quả và panel bài báo, cào dữ liệu (mock), tối ưu hiệu năng (theo commit: hơn 100 FPS trên màn hình tần số cao, LCP dưới 2.5 s), đăng nhập, chat offline và đồng bộ chat với backend.

---

## 2. Quy trình prompt đã diễn ra thế nào

### Giai đoạn 1 — Có spec: P1 → P11

Làm theo `docs/prompt_fe_3d.md` (936 dòng). Mỗi prompt có **Đặc tả → Pseudocode → Prompt gửi AI → Quy trình kiểm chứng**. Phần hướng dẫn cho AI quy định:

1. Làm tuần tự P1 → P11, không nhảy bước.
2. Kiểm chứng đạt mới sang prompt tiếp theo.
3. Ghi mọi giả định vào `ASSUMPTIONS.md`.
4. Chạy được ở chế độ mock.
5. Tách logic khỏi rendering, logic phải có unit test.
6. Cập nhật `PROGRESS.md` cuối mỗi prompt.
7. Không dùng `any` và `dangerouslySetInnerHTML`.
8. **Mọi màu lấy từ theme token, không hard-code màu trong component.**

Kết quả theo `PROGRESS.md` (2026-10-19): `vitest` 99/99, Playwright 6/6, chạy ở chế độ mock.

### Giai đoạn 2 — Bổ sung, vẫn có ghi chép: P12 → P14.1 (đến 2026-10-20)

- P12: luồng tri thức khi cào dữ liệu.
- P14.1: cáp tri thức.
- Gọn giao diện: chỉ còn theme tối; bỏ các nút camera; đưa thanh năm, nhãn nhánh và bảng cào dữ liệu vào Cài đặt.
- Hoạt cảnh tìm kiếm chậm và dịu mắt hơn.
- Unit test tăng lên 188–190, E2E 6/6.

**Vấn đề:** các prompt này có ghi trong `PROGRESS.md` nhưng **không có trong `prompt_fe_3d.md`** (spec chỉ đến P11). Người mới không có chỗ nào để đọc đặc tả của chúng.

### Giai đoạn 3 — Prompt rời rạc, không spec (2026-10-20 → 2026-10-28)

Khoảng 20 commit, nhóm theo chủ đề:

| Chủ đề | Commit | Ghi chú |
| :-- | :-- | :-- |
| Hiệu năng 3D/FPS | `199c14e`, `b368cc5`, `718b049`, `e8cb552`, `20026ee`, `0888b74`, `f6300f8` | Có báo cáo trong `frontend_rag/docs/perf-*.md` |
| Hiệu năng tải trang | `9457e1c` | LCP < 2.5 s, FCP < 450 ms |
| TopK badge | `181761e`, `86e72c5` | |
| **Auth "Liquid Glass"** | `85b3e35` → `3fdb69e` → `5871b26` → `6fb7906` → `26fa96b` → `85962c2` | **6 commit sửa đi sửa lại**: thêm, nhúng vào ChatPanel, đơn giản hoá, đổi emblem + bỏ icon, bỏ gradient + thêm lại icon, chỉnh cỡ icon |
| FPS_LOCK | `f8a483b` → `20a1c26` | Thêm rồi gỡ |
| tsconfig | `bc48609`, `90e5299` | |
| Intro / loading | `050879c`, `1ba3e20` → `538ae7f` | LoadingTree thêm rồi thay bằng WorldError |
| Chat offline + đồng bộ | `7762d3d`, `59f781f`, `d035b48` | Dexie, HistoryDrawer, đồng bộ hai chiều |

**Vấn đề của giai đoạn này:**

- Không có spec, không có bước kiểm chứng trước khi làm.
- `PROGRESS.md` không được cập nhật từ 2026-10-20. `ASSUMPTIONS.md` cũng dừng ở các giả định P1–P11 và Trig.
- Thay đổi UI diễn ra kiểu thử rồi sửa (Auth sửa 6 lần), vì không có chuẩn để đối chiếu.
- `AGENTS.md` vẫn là ghi chú Next.js cũ ("This is NOT the Next.js you know…"), trong khi dự án đã chuyển sang Vite. AI đọc file này sẽ bị dẫn sai.

### Tình trạng hiện tại

Quy trình đã **trượt từ "spec-driven" sang "prompt theo cảm hứng"**. Tài liệu (`prompt_fe_3d.md`, `PROGRESS.md`, `ASSUMPTIONS.md`, `AGENTS.md`) không còn phản ánh code.

---

## 3. Quy chuẩn thiết kế: hiện có gì, thiếu gì

### Đã có: token màu

`src/theme/tokens.ts` định nghĩa `ThemeTokens` (chỉ còn theme `dark`). `applyCssVars.ts` ghi chúng ra CSS variable trên `:root`:

| Token | Giá trị | CSS var |
| :-- | :-- | :-- |
| `uiSurface` | `#120f2499` | `--ui-surface` |
| `uiSurfaceStrong` | `#151129e6` | `--ui-surface-strong` |
| `uiBorder` | `#ffffff1f` | `--ui-border` |
| `uiInk` / `uiInkMuted` | `#ece9ff` / `#a29cc4` | `--ink` / `--ink-muted` |
| `uiAccent` / `uiAccentInk` | `#7fe8ff` / `#06121a` | `--accent` / `--accent-ink` |
| `uiWarn` / `uiDanger` / `uiSuccess` | `#ffc145` / `#ff5d73` / `#58f0a7` | `--warn` / `--danger` / `--success` |
| `categoryPalette` | 12 màu neon | `--cat-0` … `--cat-11` |

Ngoài ra `index.css` có `--font-sans` (Geist), `--font-mono` (Geist Mono) và `--radius: 14px`.

### Chưa có: mọi thứ còn lại

Không có token cho **bo góc, blur, alpha nền/viền, đổ bóng, cỡ chữ, khoảng cách, thời gian/easing của animation**. Số liệu đo trong `index.css` và các component UI (`components/{chat,layout,result,auth}`, không tính file test):

#### Bo góc: 8 mức px, cộng 999px và 50%

| Giá trị | Nơi dùng |
| :-- | :-- |
| 4px (`rounded`) | 9 lần: `HistoryDrawer.tsx`, `ChatPanel.tsx`, `PerfHUD.tsx` *(bản đầu ghi 26 lần là sai, do regex đếm nhầm cả chữ "g**rounded**")* |
| 5px | 1 lần (CSS) |
| 6px | 2 lần (CSS) |
| 8px (`rounded-lg` ×3, CSS ×1) | 4 lần |
| 10px | `.btn` và 1 chỗ khác (CSS) |
| 12px (`rounded-xl` ×6, CSS ×2) | 8 lần; có `.liquid-input-wrap`, `.liquid-cta-btn` |
| 14px | `.glass` qua `--radius`, **chỉ 1 lần** |
| 16px (`rounded-2xl` ×2, CSS ×1) | 3 lần; có `.liquid-emblem` |
| 999px / 50% / `rounded-full` | 11 lần (chip, chấm tròn) |

Panel `.glass` bo 14px, nút `.btn` bo 10px, input auth bo 12px, và có cả `!rounded-xl` đè lên `.glass` (`ChatPanel.tsx:121`). Các component đặt cạnh nhau có bo góc khác nhau.

#### Độ mờ (`backdrop-filter: blur`): 5 mức

| Giá trị | Số lần |
| :-- | :-- |
| 6px | 2 |
| 10px | 1 (`.liquid-badge-chip`) |
| 12px | 4 trong CSS + 2 `backdrop-blur-md` trong component |
| 14px | 2 (`.liquid-cta-btn`…) |
| 16px + `saturate(1.2)` | 2 (`.glass`) |

Ngoài ra còn `filter: blur(8px)`, `blur(1.5px)` và `blur(0)` cho animation.

#### Độ trong của trắng (`rgba(255,255,255,α)`): 11 mức

Các giá trị đang dùng: 0.03, 0.04, 0.05, 0.06, 0.10, 0.14, 0.15, 0.16, 0.18, 0.22, 0.25. Viền ở chỗ thì 0.14, chỗ thì 0.15 hoặc 0.18, hoặc dùng token `--ui-border` (`#ffffff1f` ≈ 0.12). Có 41 dòng trong `index.css` chứa rgba hoặc hex.

#### Cỡ chữ: khoảng 14 giá trị, trộn 3 đơn vị

- **CSS:** 10px ×4, 11px ×2, 12px, 13px, 22px, 0.68rem, 0.7rem, 0.72rem ×2, 0.8rem, 0.83rem ×2, 0.85rem, 0.9rem, 0.72em, 0.85em.
- **Component:** `text-[10px]` ×5, `text-[11px]` ×4, cộng các lớp Tailwind mặc định (`text-xs`, `text-sm`…).

#### Đổ bóng

Mỗi class tự viết `box-shadow` riêng (`.glass`, `.liquid-emblem`, `.liquid-input-wrap`, `.liquid-cta-btn`, `.liquid-badge-chip`), không có thang chung.

### Lỗi cụ thể: token và code đã lệch nhau

1. **`.glass` không dùng token `--ui-surface`.**
   - Token là `#120f2499` (alpha ≈ 0.6). `.glass` hard-code `rgba(12, 10, 24, 0.24)` (`index.css:54`).
   - `--ui-surface` được định nghĩa mà **không chỗ nào dùng**. Đổi token sẽ không làm thay đổi giao diện.
2. **Code chat mới gọi CSS variable không tồn tại.**
   - `HistoryDrawer.tsx:40`: `bg-[var(--ui-panel-bg,rgba(18,20,28,0.96))]`
   - `HistoryDrawer.tsx:91`: `var(--ui-hover,rgba(255,255,255,0.05))`, `var(--ink-secondary,#cbd5e1)`
   - `ChatPanel.tsx:39`, `ChatPanel.tsx:50`: `var(--ink-secondary,#cbd5e1)`

   `--ui-panel-bg`, `--ui-hover` và `--ink-secondary` chưa từng được định nghĩa, nên **luôn rơi về giá trị hard-code**. Ngoài ra `#cbd5e1` là màu slate của Tailwind, không thuộc bảng màu tím–neon của dự án. Như vậy AI đã viết code mà không biết bộ token đang có, đồng thời vi phạm quy tắc 8 của spec.
3. **`ChatPanel.tsx:39` dùng `!text-white`** thay vì `--accent-ink`.
3b. **Biến `--ink-faint` (dùng 5 lần) cũng chưa từng được định nghĩa.**
3c. **Dùng thẳng bảng màu mặc định của Tailwind** (`red-*`, `gray-*`, `amber-*`, `text-white`), bỏ qua token:
   - `HistoryDrawer.tsx`: 11 chỗ (`bg-red-950/80`, `border-red-500/50`, `text-gray-400`…);
   - `ChatPanel.tsx`: 4 chỗ (`bg-amber-500/15`, `text-amber-300`…);
   - `PerfHUD.tsx`: khoảng 30 chỗ. Đây là công cụ dev, có thể miễn.
4. **`--radius: 14px`** là "token bo góc" duy nhất nhưng chỉ dùng 1 lần, và không có trong `tokens.ts`.
5. **CSS chết:** `[data-theme="light"] .glass` (`index.css:64`) vẫn còn, dù theme sáng đã bị bỏ. Comment trong `tokens.ts` vẫn nhắc "pastel in light", "light theme".
6. **Hai hệ glass song song:** `.glass` (TopBar, Chat, Settings, PaperPanel, ResultCard, CategoryLegend, filter chip, bottom sheet) và `.liquid-*` (Auth, 1 chip trong ChatPanel). Hai hệ có nền, viền, blur và bo góc khác nhau.

---

## 4. Liquid Glass: đã có chưa?

**Chưa có Liquid Glass thật.**

Các class `.liquid-*` (`index.css:744–862`) gồm `.liquid-auth-shell`, `.liquid-emblem`, `.liquid-input-wrap`, `.liquid-input`, `.liquid-cta-btn` và `.liquid-badge-chip`. Chúng được dùng ở:

- `components/auth/AuthOverlay.tsx` (form đăng nhập);
- `components/chat/ChatPanel.tsx:79` (1 chip).

Bên trong chỉ là **glassmorphism cổ điển**:

- `backdrop-filter: blur(10–14px)`;
- nền trắng alpha thấp (0.03–0.10);
- viền trắng alpha 0.14–0.18;
- đường sáng `inset 0 1px 0` ở cạnh trên và cạnh dưới.

Những gì làm nên Liquid Glass (kiểu Apple) **chưa có**:

| Đặc trưng Liquid Glass | Hiện trạng |
| :-- | :-- |
| Khúc xạ: nền phía sau bị bẻ cong và phóng to ở mép kính | ❌ |
| Phản chiếu (specular highlight) di chuyển theo ánh sáng hoặc con trỏ | ❌ (chỉ có đường `inset` tĩnh) |
| Tách màu nhẹ (chromatic aberration) ở viền | ❌ |
| Hình khối "giọt nước", co giãn khi tương tác | ❌ |
| Áp dụng thống nhất cho toàn bộ UI | ❌ (chỉ Auth + 1 chip) |

### Hai hướng làm Liquid Glass thật

| Hướng | Ưu | Nhược |
| :-- | :-- | :-- |
| **SVG `feDisplacementMap` trong `backdrop-filter: url(#…)`** | Thuần CSS/SVG, gắn được vào DOM hiện có | Chỉ Chromium hỗ trợ; Safari/Firefox phải fallback về blur thường |
| **Shader trong Canvas three.js** (render panel kính như mesh/quad, lấy mẫu framebuffer của cảnh 3D) | Chạy mọi trình duyệt có WebGL; khúc xạ đúng nghĩa trên chính cảnh 3D | Tốn GPU (thêm 1 pass). Dự án vừa tối ưu để đạt hơn 100 FPS, nên **phải đo lại hiệu năng** (`npm run test:perf`, `docs/perf-report.md`) |

Dù chọn hướng nào, cũng **phải có design token trước** (mục 5, bước 1). Nếu không, Liquid Glass sẽ thành thêm một hệ glass thứ ba.

---

## 5. Đề xuất

### Bước 1 — Chốt bộ design token (trước mọi chỉnh sửa UI)

Mở rộng `src/theme/tokens.ts`, xuất ra CSS variable qua `applyCssVars.ts` và khai báo vào `@theme` của Tailwind v4 trong `index.css`. Ví dụ khung (các con số cụ thể cần nhóm và thầy chốt):

| Nhóm | Ví dụ thang |
| :-- | :-- |
| Màu | Giữ bộ hiện tại; thêm `--ink-secondary`, `--ui-hover`, `--ui-panel-bg` nếu thật sự cần, hoặc xoá các chỗ đang gọi |
| Bo góc | `--radius-sm` 6 · `--radius-md` 10 · `--radius-lg` 14 · `--radius-full` 999 |
| Blur | `--blur-sm` 8 · `--blur-md` 12 · `--blur-lg` 16 |
| Alpha nền kính | `--glass-fill-1` 0.04 · `--glass-fill-2` 0.08 · `--glass-fill-3` 0.24 |
| Alpha viền | `--glass-stroke` 0.14 · `--glass-stroke-strong` 0.22 |
| Highlight | `--glass-highlight` 0.16 |
| Đổ bóng | `--shadow-sm` · `--shadow-md` · `--shadow-lg` |
| Cỡ chữ | 10 · 11 · 12 · 13 · 14 · 16 · 22 (một đơn vị duy nhất) |
| Khoảng cách | Thang Tailwind mặc định, cấm giá trị tuỳ ý |
| Chuyển động | `--dur-fast` 120ms · `--dur-base` 180ms · `--ease-out` |

Kèm **lint rule** (ESLint hoặc stylelint) chặn `rounded-[…]`, `text-[…px]`, `blur-[…]`, hex và `rgba(` trong component.

### Bước 2 — Một component `Glass` duy nhất

- Gộp `.glass` và `.liquid-*` thành `Glass` với các biến thể `panel | card | chip | input | button`, đọc hoàn toàn từ token.
- Thay toàn bộ chỗ dùng hiện tại.
- Liquid Glass (khúc xạ, phản chiếu) được cài **bên trong** component này. Như vậy chỉ một chỗ phải đổi và một chỗ phải đo hiệu năng.

### Bước 3 — Đưa quy trình về spec-driven như P1–P11

- Thêm vào `docs/prompt_fe_3d.md`:
  - đặc tả cho P12–P14 (đã làm nhưng chưa có spec);
  - **P15 — Design System** (bước 1 và 2);
  - **P16 — Liquid Glass** (chọn hướng, ngân sách FPS, fallback).
- Thêm vào phần "Hướng dẫn dành cho AI" quy tắc: **đọc `tokens.ts` trước mọi prompt UI; không tạo CSS variable hay giá trị mới ngoài token.**
- Cập nhật `PROGRESS.md` và `ASSUMPTIONS.md` cho giai đoạn 2026-10-20 → 2026-10-28.
- Viết lại `AGENTS.md` cho đúng dự án Vite hiện tại.
- Dọn CSS chết của theme sáng.

---

## Phụ lục — Cách đo lại các số liệu trên

Chạy trong `frontend_rag/src`:

```bash
UI() { find components/chat components/layout components/result components/auth \
  -name '*.tsx' ! -name '*.test.*' -print0 | xargs -0 grep -ohE "$1" | sort | uniq -c | sort -rn; }

grep -ohE 'border-radius:\s*[^;]+' index.css | sort | uniq -c | sort -rn   # bo góc (CSS)
UI "rounded(-[a-z0-9.]+|-\[[^]]+\])?"                                       # bo góc (Tailwind)
grep -ohE '(backdrop-filter|filter):\s*[^;]+' index.css | sort | uniq -c    # blur (CSS)
UI "(backdrop-)?blur(-[a-z0-9]+|-\[[^]]+\])?"                               # blur (Tailwind)
grep -ohE 'rgba?\([^)]*\)' index.css | sort | uniq -c | sort -rn            # alpha
grep -ohE 'font-size:\s*[^;]+' index.css | sort | uniq -c | sort -rn        # cỡ chữ (CSS)
UI "text-\[[0-9.]+px\]"                                                     # cỡ chữ (Tailwind)
UI "#[0-9a-fA-F]{6,8}\b"; UI "rgba?\([^)]*\)"                               # màu hard-code trong component
```
