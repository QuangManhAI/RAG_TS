# FE_DESIGN_SYSTEM.md — Quy chuẩn thiết kế Frontend "Cây Tri Thức Vector"

- **Motivation/Background**: Giao diện `frontend_rag` đang lệch chuẩn sau nhiều vòng prompt rời rạc (8 mức bo góc, 5 mức blur, 11 mức alpha, biến CSS không tồn tại — xem [`FE_DESIGN_AUDIT.md`](FE_DESIGN_AUDIT.md)); cần một quy chuẩn chung trước khi nâng cấp Liquid Glass.
- **Purpose**: Chốt bộ design token, vật liệu kính, đặc tả component, quy tắc toán 3D (số phức/quaternion) và lộ trình Liquid Glass, dựa trên giá trị đang dùng để FE sửa ít nhất.
- **Overview Pipeline**: Đo hiện trạng bằng grep trên `src/` → chọn bậc từ giá trị dùng nhiều nhất → gộp giá trị lạc thang về bậc gần nhất → kiểm cú pháp `@theme` với Tailwind 4.3.3 của dự án.
- **Detailed Plan**: §0 Quy tắc vàng; §1 Kiến trúc token; §2 Màu; §3 Bo góc; §4 Vật liệu kính; §5 Chữ; §6 Khoảng cách/icon/lớp; §7 Chuyển động; §8 Component; §9 Toán Web3D; §10 Liquid Glass; §11 Kiểm soát; §12 Lộ trình; Phụ lục A code token; Phụ lục B bảng quy đổi.
- **References**: [`FE_DESIGN_AUDIT.md`](FE_DESIGN_AUDIT.md), [`FE3d_prompt_v2.md`](FE3d_prompt_v2.md), [`prompt_fe_3d.md`](prompt_fe_3d.md), [`frontend_rag/docs/trig-audit.md`](../frontend_rag/docs/trig-audit.md), [`frontend_rag/src/theme/tokens.ts`](../frontend_rag/src/theme/tokens.ts), [`frontend_rag/src/utils/rotation.ts`](../frontend_rag/src/utils/rotation.ts), Tailwind CSS 4.3, Apple Human Interface Guidelines (Liquid Glass).
- **Created**: 2026-10-29T22:46:54+07:00
- **Last Updated**: 2026-10-30T12:10:00+07:00

[STATUS: ACTIVE]

> **Nguyên tắc soạn quy chuẩn này: giữ cái đang có, chỉ sửa chỗ lệch.**
> - Mỗi bậc trong thang đều **lấy từ giá trị đang dùng nhiều nhất** trong code.
> - Giá trị lạc thang được gộp về bậc gần nhất, phần lớn lệch ≤ 2px hoặc ≤ 0.04 alpha nên mắt thường gần như không thấy.
> - Kiến trúc hiện tại (`tokens.ts` → `applyCssVars.ts` → CSS variable) **được giữ nguyên**, chỉ bổ sung thêm.

---

## Mục lục

0. [Quy tắc vàng](#0-quy-tắc-vàng)
1. [Kiến trúc token](#1-kiến-trúc-token)
2. [Màu sắc](#2-màu-sắc)
3. [Bo góc](#3-bo-góc)
4. [Vật liệu kính: độ mờ, độ trong, viền, highlight, bóng](#4-vật-liệu-kính)
5. [Chữ](#5-chữ)
6. [Khoảng cách, icon, lớp (z-index)](#6-khoảng-cách-icon-lớp)
7. [Chuyển động](#7-chuyển-động)
8. [Đặc tả component](#8-đặc-tả-component)
9. [Toán Web3D: số phức và quaternion thay cho sin/cos](#9-toán-web3d)
10. [Sẵn sàng nâng cấp Liquid Glass (kiểu Apple macOS/iOS 26)](#10-sẵn-sàng-liquid-glass)
11. [Kiểm soát: lint, checklist PR, quy tắc cho AI](#11-kiểm-soát)
12. [Lộ trình chuyển đổi](#12-lộ-trình-chuyển-đổi)
- [Phụ lục A — Code token sẵn để dán](#phụ-lục-a--code-token)
- [Phụ lục B — Bảng quy đổi giá trị cũ → token](#phụ-lục-b--bảng-quy-đổi)

---

## 0. Quy tắc vàng

1. **Không có con số "tự do" trong component.** Màu, bo góc, blur, alpha, cỡ chữ, bóng và thời gian chuyển động đều phải lấy từ token. Cấm:
   - hex, `rgb()`, `rgba()` viết thẳng trong `.tsx`;
   - giá trị tuỳ ý dạng `rounded-[..px]`, `text-[..px]`, `blur-[..px]`;
   - bảng màu mặc định của Tailwind (`red-500`, `gray-400`, `amber-300`, `text-white`…).
2. **Được phép** tham chiếu token bằng cú pháp tuỳ ý (đây là cách code đang làm): `text-[var(--ink-muted)]`, `bg-[color-mix(in_srgb,var(--accent)_12%,transparent)]`, nhưng tỷ lệ pha phải nằm trong thang ở §2.4.
3. **Chỉ dùng CSS variable đã khai báo.** Gọi `var(--x, fallback)` với biến không tồn tại là lỗi, không phải "phòng hờ".
4. **Mỗi bề mặt kính dùng component/class chuẩn** (§8), không tự viết `backdrop-filter`.
5. **Kính không đặt lên kính.** Phần tử nằm *bên trong* một panel kính chỉ dùng lớp tô (fill), không `backdrop-filter` thêm lần nữa (§4.1).
6. **Bo góc đồng tâm.** Bo góc trong = bo góc ngoài − padding, làm tròn về bậc gần nhất (§3.2).
7. **3D: quay bằng số phức/quaternion**, `Math.sin/cos` chỉ dùng để *tạo* rotor (§9).
8. Thêm token mới phải ghi vào file này và `ASSUMPTIONS.md` kèm lý do.

---

## 1. Kiến trúc token

```
src/theme/tokens.ts ──(applyCssVars)──► :root CSS variables ──► CSS thuần (index.css) & class Tailwind
        │                                      ▲
        │                                      └── index.css :root   (giá trị fallback cho lần vẽ đầu)
        └──► three.js / shader (đọc trực tiếp ThemeTokens, bỏ alpha bằng opaqueHex)

index.css @theme ──► thang tĩnh: bo góc, blur, cỡ chữ, easing  →  sinh class: rounded-md, backdrop-blur-lg, text-caption…
```

| Loại token | Nơi khai báo | Lý do |
| :-- | :-- | :-- |
| **Màu** (gồm màu kính có alpha) | `src/theme/tokens.ts` (giữ như hiện tại) | Dùng chung với three.js; đổi theme lúc chạy |
| **Thang tĩnh** (bo góc, blur, chữ, easing, thời gian, bóng) | `@theme` trong `src/index.css` | Không đổi theo theme. Tailwind 4 sinh class và xuất CSS variable khi biến có được dùng (đã kiểm tra với Tailwind 4.3.3) |
| **Liquid Glass** (khúc xạ, phản chiếu…) | `@theme` + `:root`, mặc định **tắt** | Bật theo cấp ở §10, không phải sửa component |

**Ba tầng đặt tên:**
- **Primitive:** giá trị thô, ví dụ `--radius-md: 10px`.
- **Semantic:** vai trò, ví dụ `--glass-stroke`, `--ink-muted`.
- **Component:** class như `.btn`, `.glass`. Component chỉ đọc token semantic, tức là chỉ đọc từ tầng dưới liền kề.

---

## 2. Màu sắc

### 2.1 Bảng màu nền tảng (giữ nguyên, đã có trong `tokens.ts`)

| Vai trò | Token TS | CSS var | Giá trị | Dùng cho |
| :-- | :-- | :-- | :-- | :-- |
| Nền trang | `bg` | `--bg` | `#070614` | Nền body, phía sau canvas |
| Chữ chính | `uiInk` | `--ink` | `#ece9ff` | Nội dung đọc |
| Chữ phụ | `uiInkMuted` | `--ink-muted` | `#a29cc4` | Mô tả, nhãn, meta |
| Nhấn | `uiAccent` | `--accent` | `#7fe8ff` | Hành động chính, focus, liên kết, trạng thái active |
| Chữ trên nền nhấn | `uiAccentInk` | `--accent-ink` | `#06121a` | Chữ và icon trên `--accent` (**không** dùng `text-white`) |
| Cảnh báo | `uiWarn` | `--warn` | `#ffc145` | Chưa xác minh, offline, lưu ý |
| Nguy hiểm | `uiDanger` | `--danger` | `#ff5d73` | Lỗi, xoá |
| Thành công | `uiSuccess` | `--success` | `#58f0a7` | Hoàn tất, đoạn được dùng |
| Bề mặt đặc | `uiSurfaceStrong` | `--ui-surface-strong` | `#151129e6` | Panel cần đọc lâu (PaperPanel, Settings) và fallback khi giảm trong suốt |
| Bề mặt vừa | `uiSurface` | `--ui-surface` | `#120f2499` | Nhãn 3D, nền `.btn` |
| Loại bài | `categoryPalette[0..11]` | `--cat-0`…`--cat-11` | 12 màu neon | Chỉ để mã hoá category, **không** dùng làm màu trang trí |

### 2.2 Bổ sung (sửa chỗ lệch)

| Token TS mới | CSS var | Giá trị | Lý do |
| :-- | :-- | :-- | :-- |
| `uiInkFaint` | `--ink-faint` | `#64607e` | Đang được dùng 5 lần mà chưa định nghĩa. Giá trị = `--ink-muted` 60% trên `--bg`. **Chỉ dùng** cho placeholder, timestamp, icon phụ, trạng thái disabled; không dùng cho chữ cần đọc (độ tương phản ≈ 3:1) |
| `glassTint` | `--glass-tint` | `#0c0a183d` (α 0.24) | Nền thật của `.glass` hiện nay đang hard-code `rgba(12,10,24,0.24)`. Đưa vào token, **giữ nguyên hình ảnh** |
| `glassFill` | `--glass-fill` | `#ffffff0a` (α 0.04) | Lớp tô trắng cho control bên trong kính (lúc nghỉ) |
| `glassFillHover` | `--glass-fill-hover` | `#ffffff14` (α 0.08) | Lớp tô khi hover/focus (thay `--ui-hover`) |
| `glassHighlight` | `--glass-highlight` | `#ffffff29` (α 0.16) | Viền sáng phía trên (specular tĩnh) |
| `glassHighlightStrong` | `--glass-highlight-strong` | `#ffffff3d` (α 0.24) | Highlight của nút CTA và trạng thái focus |
| `glassLowlight` | `--glass-lowlight` | `#ffffff0a` (α 0.04) | Viền sáng phía dưới |
| `glassStrokeStrong` | `--glass-stroke-strong` | `#ffffff38` (α 0.22) | Viền khi hover hoặc khi bật tương phản cao |

**Sửa token có sẵn:** `uiBorder` đổi `#ffffff1f` (α 0.12) → `#ffffff24` (α 0.14). Lý do: gộp hai giá trị viền đang song song (`--ui-border` 0.12 và `rgba(255,255,255,0.14)` dùng 4 lần) về một. Chênh 0.02 alpha, không nhận ra bằng mắt.

Alias trong CSS: `--glass-stroke: var(--ui-border)`.

### 2.3 Biến không tồn tại, thay bằng gì

| Đang gọi (không tồn tại) | Thay bằng |
| :-- | :-- |
| `var(--ink-secondary, #cbd5e1)` | `var(--ink-muted)` |
| `var(--ui-hover, rgba(255,255,255,0.05))` | `var(--glass-fill-hover)` |
| `var(--ui-panel-bg, rgba(18,20,28,0.96))` | class `.surface-opaque` (`--ui-surface-strong` chồng lên `--bg`, xem §4.1) — *sửa ở P17: `--ui-surface-strong` một mình (α 0.9) để lộ chữ của khung chat bên dưới* |
| `var(--ink-faint)` | giữ, nay đã được định nghĩa (§2.2) |

### 2.4 Pha màu (`color-mix`): chỉ dùng các tỷ lệ sau

Code đang pha `--accent`, `--warn`, `--danger`, `--success`, `--ink` với `transparent` theo khoảng 15 tỷ lệ khác nhau. Gộp lại thành:

| Mục đích | Tỷ lệ chuẩn | Gộp từ (đang dùng) |
| :-- | :-- | :-- |
| Nền rất nhẹ (blockquote, band) | **6%** | 5, 6, 7, 8, 9 |
| Nền nhẹ (chip, hover) | **12%** | 10, 12 |
| Nền vừa (active/pressed, bubble người dùng) | **16%** | 14, 16 |
| Nền đậm (hover của chip, highlight) | **24%** | 22, 25, 26 |
| Viền màu | **40%** | 38, 40, 45, 50 |
| Viền màu khi hover/focus | **55%** | 55, 60 |
| Glow (`box-shadow`, `text-shadow`) | **45%** / **70%** | 38–45 / 65–70 |

Ví dụ: `bg-[color-mix(in_srgb,var(--danger)_12%,transparent)]`, `border-[color-mix(in_srgb,var(--danger)_40%,transparent)]`.

### 2.5 Thay màu Tailwind mặc định

| Đang dùng | Thay bằng |
| :-- | :-- |
| `red-*` (`bg-red-950/80`, `border-red-500/50`, `text-red-300`…) | `--danger` + tỷ lệ §2.4 |
| `amber-*` (`bg-amber-500/15`, `text-amber-300`…) | `--warn` + tỷ lệ §2.4 |
| `gray-*` / `slate` (`text-gray-400`, `#cbd5e1`) | `--ink-muted` (chữ), `--ink-faint` (phụ) |
| `text-white` trên nền `--accent` | `--accent-ink` |
| `text-white` trên nền `--danger` | `--ink` |

**Ngoại lệ:** `components/world/overlay/PerfHUD.tsx` là công cụ đo dành cho dev (chỉ bật bằng `?perf`) nên được miễn quy tắc màu. Vẫn nên chuyển dần khi có dịp sửa.

### 2.6 Màu trong 3D

- three.js chỉ đọc `ThemeTokens` và bỏ alpha bằng `opaqueHex()`, không có hex trong component 3D. Quy tắc này đã có và vẫn giữ.
- Không dùng màu category cho UI trang trí. Màu category là *dữ liệu*.

---

## 3. Bo góc

### 3.1 Thang (5 bậc)

| Token | Giá trị | Class Tailwind | Dùng cho |
| :-- | :-- | :-- | :-- |
| `--radius-xs` | **4px** | `rounded-xs` | Nút icon trong dòng, `code` inline, badge rất nhỏ |
| `--radius-sm` | **6px** | `rounded-sm` | Tooltip, skeleton, điểm số 3D (`score3d`), đuôi bubble chat |
| `--radius-md` | **10px** | `rounded-md` | **Control**: button (`.btn`), field (`.field`), input, dòng danh sách, thẻ/khối nằm *trong* panel |
| `--radius-lg` | **14px** | `rounded-lg` | **Bề mặt nổi**: panel kính (`.glass`), bubble chat, emblem, nút kính đứng riêng |
| `--radius-full` | **9999px** | `rounded-full` | Pill, chip, tag, chấm tròn, thanh `.bar` |

Lý do chọn các bậc này:
- 10px (`.btn`, `.field`) và 14px (`.glass`, `--radius`) là các giá trị "xương sống" đang có.
- 4px và 6px là hai bậc nhỏ đang dùng nhiều nhất.
- Các bậc cách nhau 4px, trừ 4→6. Nhờ vậy quy tắc đồng tâm ở §3.2 khớp với padding phổ biến (`p-1` = 4px).

Thang này **thay thế** thang mặc định của Tailwind (`--radius-*: initial`, xem Phụ lục A). `rounded-xl`, `rounded-2xl`, `rounded-3xl` và `rounded` (không hậu tố) **không còn tồn tại**. Lưu ý: Tailwind không báo lỗi khi gặp class không tồn tại, chỉ lặng lẽ bỏ qua; vì vậy cần script kiểm tra ở §11.

`--radius` (14px, đang dùng ở `.glass`) giữ làm alias: `--radius: var(--radius-lg)`. Không dùng biến này trong code mới.

### 3.2 Quy tắc bo góc đồng tâm (concentric), giống Apple

Phần tử con nằm sát mép trong của phần tử cha có bán kính:

```
r_con = max(r_cha − padding, radius-xs)   → làm tròn về bậc gần nhất
```

| Cha | Padding | Con |
| :-- | :-- | :-- |
| Panel `lg` 14 | 4px (`p-1`) | `md` 10 |
| Control `md` 10 | 4px | `sm` 6 |
| Control `sm` 6 | 2px | `xs` 4 |
| Panel `lg` 14 | ≥ 12px (`p-3`) | Không ràng buộc; dùng `md` cho thẻ con |

Quy tắc này là điều kiện để §10 (Liquid Glass) trông đúng: viền khúc xạ của cha và con phải song song nhau.

---

## 4. Vật liệu kính

### 4.1 Ba cấp vật liệu (gộp `.glass`, `.glass-strong` và `.liquid-*`)

| Vật liệu | Tương ứng hiện tại | Nền | Blur | Dùng cho |
| :-- | :-- | :-- | :-- | :-- |
| **Regular** | `.glass` | `--glass-tint` (α 0.24) | `--blur-lg` 16px + `saturate(--glass-saturate)` | Bề mặt nổi trên cảnh 3D: TopBar, cột Chat, chip lọc, nút mở chat |
| **Strong** | `.glass.glass-strong` | `--ui-surface-strong` (α 0.9) | `--blur-lg` | Panel đọc lâu: PaperPanel, Settings, CategoryLegend, ResultCard, bottom sheet |
| **Clear** | `.liquid-*` | `--glass-fill` (α 0.04) | **Không blur** khi nằm trong kính khác; `--blur-md` khi đứng riêng trên cảnh 3D | Control *bên trong* kính: input, CTA, emblem, chip |

**Kính không đặt lên kính.** Form Auth và HistoryDrawer nằm *bên trong* cột Chat (đã là `.glass`). Lớp `backdrop-filter` thứ hai:
- làm đục và bẩn màu (blur chồng lên blur);
- tốn thêm một lượt lấy mẫu nền trên mỗi phần tử, đặt trên canvas WebGL đang chạy 100+ FPS.

Vì vậy control bên trong kính chỉ dùng fill + stroke + highlight. Theo HIG của Apple, Liquid Glass cũng không chồng kính lên kính.

**Strong che lên chữ thì phải đục hoàn toàn** (bổ sung ở P17). `--ui-surface-strong` có α 0.9: đủ trên cảnh 3D, nhưng một ngăn kéo phủ lên khung chat (HistoryDrawer) vẫn để lộ chữ bên dưới. Trường hợp này dùng class `.surface-opaque`: `background: linear-gradient(var(--ui-surface-strong), var(--ui-surface-strong)), var(--bg)`. Đây là hai token có sẵn chồng lên nhau, không có giá trị mới. Ở P19, `Glass material="strong" nested` phải dùng nền này.

### 4.2 Thang blur (3 bậc)

| Token | Giá trị | Class | Dùng cho |
| :-- | :-- | :-- | :-- |
| `--blur-sm` | **6px** | `backdrop-blur-sm` | Nhãn nhỏ neo trong 3D (`.label3d`, `.topk-badge`). Rẻ, có rất nhiều cái cùng lúc |
| `--blur-md` | **12px** | `backdrop-blur-md` | Control/chip đứng riêng trên cảnh 3D |
| `--blur-lg` | **16px** | `backdrop-blur-lg` | Panel (Regular, Strong) |
| `--glass-saturate` | **1.2** | — | Đi kèm `--blur-md`/`--blur-lg` (đang có ở `.glass`) |

Giá trị cũ 10px và 14px gộp về 12px. Tuy vậy, theo §4.1, phần lớn các chỗ đó (bên trong cột Chat) sẽ **bỏ blur**.

`filter: blur()` dùng cho *animation* (`dissolve` của chữ câu hỏi) là hiệu ứng chuyển động, không phải vật liệu, nên không bị ràng buộc.

### 4.3 Viền, highlight, bóng

| Token | Giá trị | Dùng cho |
| :-- | :-- | :-- |
| `--glass-stroke` (= `--ui-border`) | trắng α 0.14 | Viền 1px của mọi bề mặt kính và control |
| `--glass-stroke-strong` | trắng α 0.22 | Viền khi hover; bật tương phản cao |
| `--glass-edge` | `inset 0 1px 0 var(--glass-highlight), inset 0 -1px 0 var(--glass-lowlight)` | "Cạnh kính": highlight trên α 0.16 + dưới α 0.04 (đang có ở `.glass`) |
| `--glass-edge-strong` | `inset 0 1px 0 var(--glass-highlight-strong), inset 0 -1px 0 var(--glass-lowlight)` | CTA, control đang focus |
| `--shadow-panel` | `0 12px 36px -12px rgb(0 0 0 / 0.45)` | Bóng đổ của panel nổi (giữ từ `.glass`) |
| `--glow-accent` | `0 0 14px color-mix(in srgb, var(--accent) 45%, transparent)` | Trạng thái được chọn (score3d, topk) |

Trên kính Clear **không dùng bóng đổ**, chỉ dùng `--glass-edge`.

---

## 5. Chữ

### 5.1 Font (giữ nguyên)

- `--font-sans`: Geist cho giao diện.
- `--font-mono`: Geist Mono, kèm `tabular-nums` (class `.mono`), cho mọi con số đo được: điểm cosine, ms, FPS, ID, top-k.

### 5.2 Thang cỡ chữ (7 bậc, đơn vị px)

| Token / class | Cỡ / dòng | Dùng cho | Gộp từ (đang dùng) |
| :-- | :-- | :-- | :-- |
| `text-micro` | 10 / 14 | Meta phụ, nhãn nhánh 3D nhỏ, vòng số vector | `10px`, `text-[10px]` |
| `text-caption` | 11 / 16 | Chip, tag, badge, nhãn 3D | `11px`, `text-[11px]`, `0.68rem`, `0.7rem`, `0.72rem` |
| `text-xs` | 12 / 16 | **Mặc định cho UI dày đặc**: danh sách, pipeline, tooltip, thẻ kết quả | `12px`, `text-xs` (34 lần) |
| `text-control` | 13 / 18 | Button, field, input | `13px`, `0.8rem`, `0.83rem`, `0.85rem` |
| `text-sm` | 14 / 20 | Nội dung chat (Markdown), bubble | `0.9rem`, `text-sm` |
| `text-base` | 16 / 24 | Tiêu đề panel | `text-base` |
| `text-display` | 22 / 28 | Chữ câu hỏi bay trong 3D | `22px`, `text-2xl` |

- `.md` (Markdown) giữ `line-height: 1.65` để dễ đọc đoạn dài.
- Đơn vị `em` chỉ được dùng cho phần tử *inline* trong văn bản, cần co theo chữ cha: `code` (0.85em), `.cite-chip` (0.72em).

### 5.3 Độ đậm (4 bậc)

400 (thường) · 500 (`font-medium`: nhãn, nút) · 600 (`font-semibold`: tiêu đề, CTA, `strong`) · 700 (`font-bold`: điểm số nổi bật). Giá trị `650` trong `.md strong` gộp về 600.

---

## 6. Khoảng cách, icon, lớp

### 6.1 Khoảng cách: lưới 2px/4px của Tailwind

- **Được dùng:** `0.5 · 1 · 1.5 · 2 · 2.5 · 3 · 4 · 5 · 6 · 8` (2px → 32px). Hiện code đã gần như chỉ dùng các bậc này: `gap-2` 42 lần, `gap-1` 21 lần, `px-3`/`py-2` mỗi cái 12 lần.
- `gap-3.5`, `px-3.5`: đổi về `3` hoặc `4`.
- CSS thuần: padding và gap là bội số của 2px. Ví dụ `0.4rem` (6.4px) → `6px`, `0.45rem` → `8px`, `0.55rem` → `8px`, `0.7rem` → `12px`.

### 6.2 Icon (lucide)

Quy tắc: **cỡ icon = cỡ chữ đi kèm + 2px**, làm tròn về bậc trong bảng (chốt ở P18: quy tắc này thắng cột "gộp từ").

| Bối cảnh | Cỡ | Gộp từ |
| :-- | :-- | :-- |
| Đi với `text-micro` / `text-caption` | **12** | 10, 12 (và 13 khi chữ đi kèm ≤ 11px) |
| Đi với `text-xs` / `text-control`, icon trong nút | **14** | 13, 14, 15 |
| TopBar, nút ở header của panel (Chat, Settings, PaperPanel), tiêu đề `text-sm` | **16** | 13, 15, 16 |
| Emblem | **28** | 28 |

Nét icon: mặc định `strokeWidth` 2. Riêng 12px dùng 2.25 để không bị mảnh.

### 6.3 Lớp (z-index): đặt nghĩa cho các số đang dùng

| Lớp | Class | Nội dung |
| :-- | :-- | :-- |
| Cảnh | `z-0` | Canvas 3D |
| Overlay 3D | `z-10` | Nhãn DOM neo trong 3D, ResultCard |
| Chrome | `z-20` | TopBar, cột Chat, bottom sheet, legend |
| Ngăn kéo | `z-30` | HistoryDrawer, PaperPanel |
| Hộp thoại | `z-40` | Settings, Auth, xác nhận xoá, toast |
| Dev | `z-50` | PerfHUD |

`z-[1]` (1 chỗ) đổi về `z-10`.

---

## 7. Chuyển động

### 7.1 Thời gian (4 bậc UI + 1 bậc cảnh)

| Token | Giá trị | Dùng cho | Gộp từ |
| :-- | :-- | :-- | :-- |
| `--dur-press` | **120ms** | `transform` khi nhấn | 0.1s, 0.12s |
| `--dur-hover` | **160ms** | Đổi màu, nền, viền khi hover/focus | 0.15s, 0.16s, 0.18s, 0.2s |
| `--dur-enter` | **240ms** | Xuất hiện/biến mất: fade, slide panel, tooltip, di chuyển badge | 0.24s, 0.25s, 0.3s, 0.35s |
| `--dur-reveal` | **480ms** | Thẻ/badge "bật" ra, intro, thanh điểm | 0.45s, 0.5s, 0.6s |
| `--dur-scene` | **1500ms** | Cột Chat trượt, đồng bộ với camera (`CHAT_SLIDE_*` trong `CameraDirector`) | 1.5s, **phải khớp hằng số trong JS** |

Animation lặp (`pulse`, `blink`, `shimmer`, `twinkle`, `grow`) giữ nhịp riêng, vì đó là "nhịp thở" chứ không phải chuyển trạng thái.

### 7.2 Easing (3 đường cong, đều đã có)

| Token / class | Giá trị | Dùng cho |
| :-- | :-- | :-- |
| `--ease-standard` / `ease-standard` | `cubic-bezier(0.2, 0.9, 0.3, 1)` | Mặc định cho mọi chuyển trạng thái. `(0.2, 0.8, 0.3, 1)` gộp vào đây |
| `--ease-spring` / `ease-spring` | `cubic-bezier(0.2, 0.9, 0.3, 1.2)` | Phần tử "bật" ra có vượt nhẹ (badge, ResultCard) |
| `--ease-scene` / `ease-scene` | `cubic-bezier(0.65, 0, 0.35, 1)` | Chuyển động lớn đồng bộ với camera |

### 7.3 Giảm chuyển động (giữ nguyên, đã đúng)

`prefers-reduced-motion` và `[data-reduce-motion="true"]` đưa mọi animation/transition về 0.001ms. Mọi hiệu ứng mới (kể cả Liquid Glass ở §10) phải tôn trọng hai điều kiện này.

---

## 8. Đặc tả component

### 8.1 Bảng chuẩn

| Component | Bo góc | Vật liệu | Padding | Chữ | Ghi chú |
| :-- | :-- | :-- | :-- | :-- | :-- |
| Panel (`.glass`) | `lg` | Regular | `p-3`–`p-4` | — | Bóng `--shadow-panel`, cạnh `--glass-edge` |
| Panel đọc lâu (`.glass.glass-strong`) | `lg` | Strong | `p-3`–`p-4` | — | |
| Button (`.btn`) | `md` | Nền `--ui-surface` 60%, viền `--glass-stroke` | `6px 12px` | `text-control`, 500 | |
| Button chính (`.btn-primary`) | `md` | Nền `--accent` | như trên | `--accent-ink`, 600 | **Chỉ hành động chính mới được tô màu nhấn** (giống Apple: tint chỉ dành cho hành động chính) |
| Button icon (`.btn-icon`) | `md` | như `.btn` | `6px` | — | Icon 14 |
| Field / input (`.field`, input Auth) | `md` | Clear | `8px 10px` | `text-control` | Focus: viền `--accent` 55% + `--glass-edge-strong` |
| CTA trong kính (Auth) | `md` | Clear + `--glass-edge-strong` | `10px 16px` | `text-control`, 600 | |
| Chip / tag (`.tag`, `.liquid-badge-chip`) | `full` | Clear | `2px 8px` | `text-caption` | |
| Chip trích dẫn (`.cite-chip`) | `full` | Nền accent 12%, viền accent 40% | `0 6px` | `0.72em` mono | Chưa xác minh: viền đứt `--warn` |
| Tooltip | `sm` | Strong | `4px 8px` | `text-xs` | |
| Bubble chat | `lg`, đuôi `sm` | Người dùng: accent 16%; trợ lý: ink 6% | `8px 12px` (`px-3`, §6.1 không có bậc 3.5) | `text-sm` | |
| Thẻ trong panel (chunk, alert) | `md` | Viền `--glass-stroke` | `8px 12px` | `text-xs` | |
| Dòng danh sách (History, Legend) | `md` | Hover `--glass-fill-hover` | `6px 10px` | `text-xs` | |
| Nhãn 3D (`.label3d`, `.topk-badge`) | `full` | `--ui-surface`, `--blur-sm` | `2px 8px` | `text-caption`/`text-micro` mono | |

### 8.2 Trạng thái tương tác (thống nhất mọi control)

| Trạng thái | Quy tắc |
| :-- | :-- |
| Hover | Nền lên một bậc (`--glass-fill` → `--glass-fill-hover`, hoặc accent 12%); viền accent 55%; `--dur-hover` |
| Nhấn | `translateY(1px)`; `--dur-press` |
| Đang bật (`aria-pressed`) | Viền `--accent`, chữ `--accent`, nền accent 16% |
| Focus bàn phím | `outline: 2px solid var(--accent); outline-offset: 2px` (đã có, giữ nguyên) |
| Disabled | `opacity: 0.45; cursor: not-allowed` (đã có, giữ nguyên) |
| Lỗi (`aria-invalid`) | Viền `--danger` |

### 8.3 Component `Glass` (đích đến để nâng cấp Liquid Glass)

Gộp `.glass`, `.glass-strong` và `.liquid-*` thành **một** component, như vậy §10 chỉ phải sửa ở một chỗ:

```tsx
// src/components/ui/Glass.tsx
type GlassProps = {
  material?: 'regular' | 'strong' | 'clear'; // §4.1
  shape?: 'panel' | 'control' | 'chip';      // → radius lg | md | full
  nested?: boolean;                          // true = nằm trong kính khác → không backdrop-filter (§4.1)
  interactive?: boolean;                     // bật hover/press + specular theo con trỏ (§10)
  as?: keyof JSX.IntrinsicElements;
} & React.HTMLAttributes<HTMLElement>;
```

Xuất ra DOM thống nhất:

```html
<div class="glass" data-material="regular" data-shape="panel" data-nested="false">
  <!-- ::before = lớp nền/khúc xạ, ::after = lớp specular/cạnh (§10.4) -->
  ...nội dung...
</div>
```

Giữ class `.glass` và `.glass-strong` cũ làm alias trong thời gian chuyển đổi, để không phải sửa tất cả component cùng lúc.

**Ghi chú triển khai (P19):**
- `position: relative; isolation: isolate` của `.glass` nằm trong `@layer components`, để class tiện ích `absolute`/`fixed` trên cùng phần tử vẫn thắng (bottom sheet, tooltip 3D).
- Phần sơn (viền, bo góc, bóng, `::before`, `::after`) để ngoài layer như các class khác trong `index.css`. Vì vậy muốn ghi đè bo góc phải dùng `!` (ví dụ `!rounded-b-none`).
- CTA chính trên kính dùng thêm class `.glass-cta` (viền `--glass-stroke-strong`, cạnh `--glass-edge-strong`).
- Ngăn kéo phủ chữ (HistoryDrawer) **không** dùng `Glass`, chỉ dùng `.surface-opaque`. Nó là lớp phủ đục, không phải bề mặt kính.

---

## 9. Toán Web3D

### 9.1 Nguyên tắc

> **Phép quay được biểu diễn bằng số phức (2D) hoặc quaternion (3D), không bằng góc.**
> `Math.cos/Math.sin` chỉ được gọi để **tạo** rotor (một lần, ở module scope hoặc `useMemo`), không gọi lặp lại trong vòng lặp hay trong mỗi frame.

Lý do:
- **Hợp nhiều phép quay chỉ là phép nhân.** Nhân hai số phức tốn 4 phép nhân + 2 phép cộng, thay vì cộng góc rồi gọi lại `sin`/`cos`.
- **Không có gimbal lock.**
- **Nội suy hướng đúng** bằng `slerp`.
- **Sinh N điểm đều trên vòng tròn** chỉ cần 1 cặp `cos/sin` thay vì N cặp.

Thư viện chuẩn đã có sẵn và đã có test: **`src/utils/rotation.ts`**.

| Hàm | Ý nghĩa |
| :-- | :-- |
| `makeRotor(θ)` | e^{iθ} = cos θ + i·sin θ. **Đây là chỗ duy nhất gọi lượng giác** |
| `rotateComplex(x, z, r)` | Quay điểm (x + iz) trên mặt phẳng xz |
| `mulRotor(a, b)` | Hợp hai phép quay (góc cộng nhau) |
| `stepRotation(state, r)` | Quay một bước, in-place, không cấp phát. Tự chuẩn hoá lại mỗi `RENORM_EVERY` = 64 bước |
| `makeFixedStepper` / `advanceStepper` | Quay liên tục theo thời gian với bước cố định, không phụ thuộc FPS |
| `slerpQuat(a, b, t, out)` | Nội suy hướng 3D (quaternion) |

### 9.2 Quy tắc theo tình huống

| Tình huống | Bắt buộc | Cấm |
| :-- | :-- | :-- |
| Quay vector/điểm một góc cố định | Rotor hằng ở module scope + `rotateComplex` | Viết tay `x*cos(a) - z*sin(a)` |
| Sinh N điểm đều trên vòng tròn (tiết diện ống, vòng) | 1 rotor bước `makeRotor(2π/N)`, nhân dồn | Gọi `cos/sin` cho từng điểm |
| Hợp nhiều phép quay (ví dụ vòng + xoắn) | `mulRotor` / `Quaternion.multiply` | Cộng góc rồi gọi lại `sin/cos` |
| Lấy hướng từ dữ liệu (x, z) | Chuẩn hoá `(x, z)/‖·‖`, chính nó **là** rotor | `atan2` rồi `cos/sin` lại |
| Điểm ngẫu nhiên đều trong hình tròn | Lấy mẫu loại bỏ (rejection sampling), §9.3 | `sqrt(rng()) · (cos a, sin a)` |
| Hướng của vật 3D | `object.quaternion` + `slerpQuat` | `object.rotation.set(x, y, z)` (Euler) theo thời gian |
| Quay liên tục theo thời gian | `makeFixedStepper` + `advanceStepper` trong `useFrame` | Cộng dồn góc rồi gọi `sin/cos` |
| Cùng một góc cho mọi đỉnh trong shader | Tính rotor trên CPU, truyền `uniform vec2 uRotor` | Gọi `cos/sin(uAngle)` cho từng đỉnh |
| Trong `useFrame` | Rotor/vector tạo sẵn, sửa in-place | `new Vector3()` / tạo rotor mới mỗi frame |

**Ngưỡng chính xác:** sai lệch vị trí < **1e-3** đơn vị thế giới so với công thức `sin/cos` gốc. Ngưỡng này lấy từ trig-audit.

### 9.3 Mẫu code

**Quay một góc cố định:** sửa `cameraPresets.ts:57–59`.

```ts
// module scope — gọi lượng giác một lần khi load
const FOCUS_ROTOR = makeRotor((35 * Math.PI) / 180);
// ...
const [rx, rz] = rotateComplex(hx, hz, FOCUS_ROTOR);
```

**Vòng tròn N điểm bằng nhân dồn rotor:** sửa `treeGeometry.ts:66–68`.

```ts
const step = makeRotor((2 * Math.PI) / radial);
let re = 1, im = 0;                       // e^{i·0}
for (let j = 0; j <= radial; j++) {
  const s = im, c = -re;                  // giữ đúng quy ước cũ: s = sin a, c = −cos a
  // ... dùng s, c như trước ...
  const nre = re * step.re - im * step.im;
  im = re * step.im + im * step.re;
  re = nre;
}
```

Sai số tích luỹ khoảng N·1e-16, với N ≤ 64 thì không cần chuẩn hoá lại. Nếu N > 64 thì dùng `stepRotation`.

**Vòng + xoắn:** `treeGeometry.ts:144–146` (`cos(a + twist)`). Lấy rotor của hàng `makeRotor(twist)` nhân với rotor vòng bằng `mulRotor`.

**Hướng từ dữ liệu, không dùng `atan2`:** `mockWorld.ts:89–95`.

```ts
const h = Math.hypot(px, pz);
const base: Rotor = { re: px / h, im: pz / h };   // chính là e^{i·atan2(pz, px)}
const dir = mulRotor(base, OFFSET_ROTOR);
```

**Điểm ngẫu nhiên trong hình tròn, không lượng giác, không `sqrt`:** `FeedCables.tsx:86–90`, `:173–176`.

```ts
/** Uniform point in the unit disk — same distribution as sqrt(u)·(cos 2πv, sin 2πv). */
function randomInDisk(rng: () => number, out: [number, number]): [number, number] {
  let x: number, y: number;
  do { x = rng() * 2 - 1; y = rng() * 2 - 1; } while (x * x + y * y > 1);
  out[0] = x; out[1] = y;
  return out;
}
```

Trung bình cần 1.27 lần thử (2.55 lần gọi `rng`). Lưu ý: cách này thay đổi chuỗi số ngẫu nhiên đã dùng, nên **hình ảnh khác nhưng cùng phân bố**. Nếu có test so khớp với seed cố định thì phải cập nhật giá trị kỳ vọng.

**Rotor đồng nhất trong shader:**

```glsl
uniform vec2 uRotor;                      // (cos θ, sin θ) tính trên CPU mỗi khi θ đổi
vec2 rot(vec2 p) { return vec2(p.x * uRotor.x - p.y * uRotor.y, p.x * uRotor.y + p.y * uRotor.x); }
```

### 9.4 Ngoại lệ được phép dùng `sin/cos` (có số đo trong `trig-audit.md`)

| Trường hợp | Lý do giữ |
| :-- | :-- |
| **Dao động vô hướng theo thời gian** cần đúng pha: quả cầu "thở" (`QueryFlight.tsx:331`), ô đất nhấp sáng (`GroundGrid.tsx:153`) | Đây là *dao động* chứ không phải *quay*. Đã thử rotor + bước cố định: nhanh hơn 11 ns/frame ở một chỗ, nhưng **chậm hơn** 1.08–1.31× ở chỗ còn lại, và lệch pha làm sai 3.4e-3 (vượt ngưỡng 1e-3). Frame time không đổi nên đã hoàn tác |
| **Trong shader GLSL** khi góc khác nhau cho mỗi đỉnh/điểm (lá nhấp nháy, đom đóm, gân cây) | GPU tính `sin` bằng 1 lệnh SFU. Đo cho thấy shader không phải nút thắt |
| **Hàm tuần hoàn dùng làm nhiễu** (`bark()` trong `treeGeometry.ts:104–106`, `flare`) | `sin(k·a + …)` ở đây là hàm sóng tạo vân vỏ cây, không phải phép quay |
| `Math.tan(fov/2)` (`materials.ts:15`) | Đổi kích thước điểm sang pixel, không phải phép quay |
| Bên trong `makeRotor` | Chỗ hợp lệ duy nhất để tạo rotor |

Muốn bỏ một ngoại lệ, phải đo lại theo phương pháp ở `trig-audit.md` §1 (microbenchmark `npm run bench` + frame time 600 frame), và chỉ đổi khi đạt **≥ 5% cải thiện tại chỗ** mà **sai lệch < 1e-3**.

### 9.5 Hiện trạng các chỗ gọi lượng giác so với quy chuẩn

| Vị trí | Nhóm | Theo chuẩn | Việc cần làm |
| :-- | :-: | :-- | :-- |
| `utils/cameraPresets.ts:57–59` | A | Quay góc cố định | **Đổi** sang `FOCUS_ROTOR` + `rotateComplex` |
| `utils/cameraPresets.ts:30` (pitch overview) | A | Quay góc cố định | **Đổi** sang `PITCH_ROTOR` (re = cos, im = sin) |
| `utils/treeGeometry.ts:66–68` | A | Vòng N điểm | **Đổi** sang nhân dồn rotor |
| `utils/treeGeometry.ts:144–146` | A | Vòng + xoắn | **Đổi** sang `mulRotor` |
| `utils/treeGeometry.ts:104–106, 141` | A | Nhiễu tuần hoàn | Giữ (ngoại lệ) |
| `components/world/fx/Fireflies.tsx:45` | A | Điểm ngẫu nhiên trên vòng | **Đổi** sang `randomInDisk` chuẩn hoá, hoặc giữ vì chỉ chạy lúc mount (ưu tiên thấp) |
| `components/world/fx/FeedCables.tsx:90, 176` | B/C | Điểm ngẫu nhiên trong hình tròn, **chạy lúc phát hạt** | **Đổi** sang `randomInDisk` (ưu tiên cao nhất vì chạy lúc runtime) |
| `utils/feedCables.ts:102–103` | A | Hướng mặc định khi r ≤ 1 | **Đổi**: dùng rotor tạo từ góc đó |
| `api/mock/mockWorld.ts:76, 89–95, 158–159` | A | `atan2` → `cos/sin` | **Đổi** sang chuẩn hoá vector + `mulRotor` (mock, ưu tiên thấp) |
| `components/world/fx/QueryFlight.tsx:331` | B | Dao động | Giữ (ngoại lệ đã đo) |
| `components/world/ground/GroundGrid.tsx:153` | C | Dao động | Giữ (ngoại lệ đã đo) |
| `components/world/materials.ts:15` | B | `tan` FOV | Giữ (ngoại lệ) |
| Shader (`canopyShader.ts`, `Fireflies`, `QueryText`, `Branches`) | D | Góc khác nhau theo đỉnh | Giữ; góc đồng nhất thì dùng `uRotor` |

Góc Euler: hiện **không có** chỗ nào dùng `rotation.x|y|z` theo thời gian. Quy chuẩn này giữ cho điều đó tiếp tục đúng.

**Trạng thái (P20, 2026-10-30):** mọi chỗ "Đổi" ở bảng trên đã chuyển sang rotor. Thêm 3 helper `ringRotors`, `powRotor`, `randomDirection` vào `rotation.ts`. Không chỗ nào chậm hơn (ống 2.96×, thân 1.57×, đĩa 1.65× nhanh hơn), sai lệch tối đa 1.4e-14. `npm run lint` chặn lượng giác không có chú thích. Chi tiết ở [`frontend_rag/docs/trig-audit.md`](../frontend_rag/docs/trig-audit.md) §7.

Các chỗ thuộc nhóm A chạy một lần lúc dựng cảnh, nên đổi sang rotor là để **code thống nhất**, không kỳ vọng tăng FPS. Đổi xong phải chạy lại `treeGeometry.test.ts` và chụp ảnh so sánh trước/sau.

---

## 10. Sẵn sàng Liquid Glass

### 10.1 Liquid Glass của Apple gồm những gì

Đây là ngôn ngữ thiết kế Apple giới thiệu cho macOS/iOS 26 (WWDC 2025). Các đặc trưng chính và cách quy chuẩn này chuẩn bị cho từng đặc trưng:

| Đặc trưng | Mô tả | Quy chuẩn này chuẩn bị gì |
| :-- | :-- | :-- |
| **Lensing (khúc xạ)** | Nền phía sau bị bẻ cong và phóng nhẹ ở dải mép kính, như qua một thấu kính dày | Token `--lg-refraction`, `--lg-bezel`; lớp `::before` riêng (§10.4) |
| **Specular động** | Vệt sáng trên cạnh kính di chuyển theo nguồn sáng/chuyển động | `--glass-edge` (tĩnh, đã có) + `--lg-light-x/y` (hướng sáng, §10.3) |
| **Thích ứng nền** | Độ đậm của tint và bóng tự đổi theo độ sáng nội dung phía sau | Hai vật liệu Regular và Strong; Clear dùng `--lg-dim` khi nền sáng |
| **Hai biến thể Regular / Clear** | Regular mờ và dễ đọc; Clear trong hơn, cần lớp làm tối khi nền sáng | Vật liệu Regular / Clear ở §4.1 |
| **Tầng chức năng** | Kính chỉ dành cho control/điều hướng nổi trên nội dung, không dùng cho chính nội dung | Quy tắc §10.2 |
| **Không chồng kính** | Kính không nằm trên kính | Quy tắc §4.1 (`nested`) |
| **Hình dạng đồng tâm** | Bo góc con đồng tâm với cha | §3.2 |
| **Tint chỉ cho hành động chính** | Chỉ nút chính được tô màu | §8.1 (`.btn-primary`) |
| **Morph / hợp nhất** | Các khối kính gần nhau hoà vào nhau khi chuyển trạng thái | Dự phòng ở Cấp 4 (§10.5) |
| **Trợ năng** | Reduce Transparency → kính đục; Increase Contrast → viền rõ; Reduce Motion → không đàn hồi | §10.6 |

### 10.2 Kính dùng ở đâu, không dùng ở đâu

- **Dùng (tầng control nổi trên cảnh 3D):** TopBar, cột Chat (khung), chip lọc, nút mở chat, nhãn và badge 3D, thanh công cụ, các nút trong Settings.
- **Không dùng (nội dung):** đoạn Markdown, danh sách chunk, bảng điểm, nội dung PaperPanel. Phần này nằm trên **Strong** hoặc nền đặc để chữ luôn dễ đọc.

### 10.3 Token Liquid Glass (mặc định tắt, nên không làm đổi giao diện hiện tại)

| Token | Mặc định (Cấp 1) | Gợi ý khi bật | Ý nghĩa |
| :-- | :-- | :-- | :-- |
| `--lg-level` | `1` | `2`–`4` | Cấp hiện hành (§10.5), JS ghi vào `data-lg-level` trên `<html>` |
| `--lg-refraction` | `0px` | `12px`–`24px` | Độ lệch tối đa của nền ở dải mép (scale của displacement) |
| `--lg-bezel` | `0px` | bằng `--radius` của phần tử | Bề rộng dải mép có khúc xạ |
| `--lg-chroma` | `0px` | `0.5px`–`1px` | Tách màu R/B ở mép (chromatic aberration) |
| `--lg-specular` | `0` | `0.35` | Cường độ vệt sáng động (alpha) |
| `--lg-specular-size` | `60%` | `40%`–`70%` | Bán kính vệt sáng |
| `--lg-light-x`, `--lg-light-y` | `0`, `-1` | JS cập nhật | **Hướng nguồn sáng: vector đơn vị (re, im)**. Là rotor, quay bằng `rotateComplex` theo con trỏ/con quay hồi chuyển, không lưu góc (§9) |
| `--lg-dim` | `0` | `0.2` | Lớp làm tối dưới chữ khi vật liệu Clear nằm trên nền sáng |

### 10.4 Cấu trúc lớp của một bề mặt kính (cố định ngay từ bây giờ)

```
.glass (position: relative; isolation: isolate; border-radius: var(--r))
├── ::before   z -1 : LỚP NỀN — backdrop-filter: blur() saturate()  [+ url(#lg-refract) ở Cấp 3]
│                     background: tint của vật liệu
├── nội dung   z  0
└── ::after    z  1 : LỚP CẠNH/SPECULAR — box-shadow: var(--glass-edge)
                      background: radial-gradient(at <hướng sáng>, rgb(255 255 255 / var(--lg-specular)), transparent var(--lg-specular-size))
                      pointer-events: none; mix-blend-mode: plus-lighter
```

**Vì sao làm từ bây giờ:**
- Đưa `backdrop-filter` xuống `::before` giúp Cấp 3 chỉ phải *thêm* một `url(#…)` vào filter, không phải đụng vào component.
- Đưa specular vào `::after` giúp Cấp 2 chỉ cần đổi `--lg-specular` từ 0 lên.
- Ở Cấp 1, hình ảnh giống hệt `.glass` hiện tại.

### 10.5 Các cấp nâng cấp (bật dần, mỗi cấp có fallback xuống cấp dưới)

| Cấp | Tên | Kỹ thuật | Trình duyệt | Chi phí |
| :-: | :-- | :-- | :-- | :-- |
| **0** | Đục | `--ui-surface-strong`, không blur | Mọi nơi | Thấp nhất |
| **1** | Kính mờ (**hiện tại**) | `backdrop-filter: blur() saturate()` + `--glass-edge` | Mọi trình duyệt hiện đại | Thấp |
| **2** | Kính + specular động | Cấp 1 + gradient ở `::after` theo `--lg-light-x/y` (JS cập nhật bằng rotor theo con trỏ, gom theo `requestAnimationFrame`) | Mọi nơi | Rất thấp (chỉ đổi CSS variable) |
| **3** | Khúc xạ SVG | Cấp 2 + `backdrop-filter: url(#lg-refract)` với `<feDisplacementMap>`. Bản đồ dịch chuyển sinh từ `--radius` và `--lg-bezel` (SDF của hình chữ nhật bo góc). Khi `--lg-chroma > 0` thì lệch R/B riêng | **Chỉ Chromium.** Safari và Firefox tự rơi về Cấp 2 | Trung bình: mỗi bề mặt thêm một lượt lọc |
| **4** | Kính WebGL | Panel lớn trên canvas được vẽ trong R3F: một quad màn hình lấy mẫu render target của cảnh. Pháp tuyến = gradient SDF bo góc; độ lệch = pháp tuyến × profile mép × `--lg-refraction`; tách màu bằng cách lấy mẫu R/G/B lệch nhau. Hỗ trợ morph giữa các khối | Mọi nơi có WebGL2 | Cao: thêm một pass, **phải đo** |

**Phát hiện tính năng:**
- Cấp 3: `CSS.supports('backdrop-filter', 'url(#a)')` chưa đáng tin trên mọi trình duyệt, nên kết hợp với kiểm tra engine Chromium.
- Cấp 4: dựa vào `AdaptiveQuality` hiện có. Chất lượng "Thấp" thì tối đa Cấp 2.

**Ngân sách hiệu năng (bắt buộc trước khi bật Cấp 3 hoặc 4):**
- Đo bằng `npm run test:perf` và theo phương pháp trong `frontend_rag/docs/perf-report.md`, cùng kịch bản và cùng máy.
- Frame time median tăng không quá **10%** so với baseline Cấp 1. Con số 10% là đề xuất, cần nhóm chốt.
- Cùng lúc tối đa **4** bề mặt có `backdrop-filter` lớn, và tối đa **1** bề mặt khúc xạ lớn (cột Chat).
- Nhãn 3D (`.label3d`, `.topk-badge`) **không bao giờ** khúc xạ, vì số lượng nhiều và chạy theo từng frame.

### 10.6 Trợ năng (map với cài đặt của hệ điều hành)

| Điều kiện | Hành vi |
| :-- | :-- |
| `@media (prefers-reduced-transparency: reduce)` (nếu trình duyệt hỗ trợ) hoặc tuỳ chọn trong Settings | Ép **Cấp 0** |
| `@media (prefers-contrast: more)` | Viền `--glass-stroke-strong`; tắt khúc xạ; `--lg-dim` lên 0.35 |
| `prefers-reduced-motion` / `data-reduce-motion` | Specular đứng yên (không theo con trỏ), không morph |
| Chữ trên kính Clear | Độ tương phản ≥ 4.5:1 so với nền *thực tế* phía sau. Không đạt thì bật `--lg-dim` hoặc chuyển sang Regular |

---

## 11. Kiểm soát

### 11.1 Script kiểm tra (đề xuất `frontend_rag/scripts/check-design.cjs`, chạy trong `npm run lint` và CI)

Báo lỗi khi file `.tsx` trong `src/components` (trừ `*.test.*` và `PerfHUD.tsx`) có:

| Mẫu bị cấm | Regex gợi ý |
| :-- | :-- |
| Hex / rgb trong component | `#[0-9a-fA-F]{3,8}\b`, `rgba?\(` |
| Màu Tailwind mặc định | `\b(bg\|text\|border\|ring)-(white\|black\|(red\|amber\|gray\|slate\|neutral\|…)-\d{2,3})` |
| Giá trị tuỳ ý thô | `(rounded\|text\|blur\|backdrop-blur)-\[\d` |
| Class bo góc đã bỏ | `\brounded(-(xl\|2xl\|3xl))?\b(?!-)` |
| CSS var không có trong danh sách token | `var\(--([a-z0-9-]+)` không nằm trong danh sách xuất từ `tokens.ts` + `@theme` |
| Lượng giác trong `useFrame` | `Math\.(sin\|cos)\(` trong file có `useFrame`, trừ các dòng ngoại lệ ở §9.4 (đánh dấu bằng comment `// trig-ok: <lý do>`) |

Trong `index.css`: ngoài khối `:root`/`@theme`, không có `border-radius`, `backdrop-filter`, `font-size` hay `rgba(` với giá trị thô.

### 11.2 Checklist PR giao diện

- [ ] Không có giá trị thô (§0). Script kiểm tra pass.
- [ ] Bo góc đúng vai trò (§3.1) và đồng tâm (§3.2).
- [ ] Bề mặt kính dùng `Glass` hoặc `.glass`; không có kính chồng lên kính.
- [ ] Chữ trên kính đạt độ tương phản (§10.6).
- [ ] Trạng thái hover / nhấn / focus / disabled theo §8.2.
- [ ] Animation dùng `--dur-*` và `ease-*`; giảm chuyển động hoạt động.
- [ ] Ảnh chụp trước/sau (dark, `overview`) đính kèm PR.
- [ ] Nếu đụng 3D: có số đo frame time trước/sau.

### 11.3 Quy tắc thêm vào "Hướng dẫn dành cho AI" trong `prompt_fe_3d.md`

> 9. **Trước mọi prompt giao diện, đọc `docs/FE_DESIGN_SYSTEM.md` và `src/theme/tokens.ts`.** Chỉ dùng token đã có. Không tạo CSS variable, màu, bo góc, blur hay cỡ chữ mới. Nếu thật sự cần token mới, dừng lại và ghi vào `ASSUMPTIONS.md` để người duyệt quyết định.
> 10. **3D: phép quay dùng `src/utils/rotation.ts`** (số phức/quaternion). `Math.sin/cos` chỉ được dùng trong các ngoại lệ ở §9.4, kèm comment `// trig-ok: <lý do>`.
> 11. Cuối mỗi prompt giao diện: chạy script kiểm tra thiết kế và đính kèm ảnh chụp trước/sau.

---

## 12. Lộ trình chuyển đổi

Mỗi bước là một PR riêng, đều nhỏ và dễ hoàn tác.

| Bước | Nội dung | Ảnh hưởng hình ảnh | Kiểm chứng |
| :-: | :-- | :-- | :-- |
| 1 | Thêm token (Phụ lục A) vào `tokens.ts`, `applyCssVars.ts`, `index.css`. Chưa sửa component nào | **Không** (trừ `--radius-*: initial` thì để đến bước 3) | `tokens.test.ts`, build |
| 2 | Sửa lỗi: định nghĩa `--ink-faint`; thay `--ink-secondary`, `--ui-hover`, `--ui-panel-bg`; thay màu Tailwind trong `HistoryDrawer.tsx` và `ChatPanel.tsx`; `.glass` dùng `--glass-tint`; xoá CSS `[data-theme="light"]` và comment theme sáng | Rất nhỏ (xám slate → tím xám của dự án) | Ảnh trước/sau, unit + E2E |
| 3 | Quy đổi bo góc, blur, cỡ chữ, alpha và thời gian theo Phụ lục B; bật `--radius-*: initial`; bỏ blur của phần tử nằm trong kính | ≤ 2px / ≤ 0.04 alpha | Script kiểm tra pass, ảnh trước/sau |
| 4 | Component `Glass` với cấu trúc lớp `::before`/`::after` (§10.4); chuyển dần các chỗ dùng | Không (Cấp 1) | Ảnh trước/sau giống nhau |
| 5 | Web3D: đổi các chỗ được đánh dấu "Đổi" ở §9.5. Ưu tiên `FeedCables` (chạy lúc runtime) | Không (sai lệch < 1e-3); riêng `randomInDisk` cho phân bố giống nhưng chuỗi ngẫu nhiên khác | `treeGeometry.test.ts`, `npm run bench`, frame time |
| 6 | Liquid Glass Cấp 2 (specular động) | Có, đây là nâng cấp | Ảnh, trợ năng §10.6 |
| 7 | Liquid Glass Cấp 3 / 4 | Có | Ngân sách hiệu năng §10.5 |

---

## Phụ lục A — Code token

### A.1 `src/theme/tokens.ts` (thêm vào `ThemeTokens` và `darkTheme`)

```ts
  /** Faint ink: placeholders, timestamps, disabled — never body text (~3:1). */
  uiInkFaint: string;
  /** Glass material (see docs/FE_DESIGN_SYSTEM.md §4). */
  glassTint: string;
  glassFill: string;
  glassFillHover: string;
  glassHighlight: string;
  glassHighlightStrong: string;
  glassLowlight: string;
  glassStrokeStrong: string;
```

```ts
  uiBorder: '#ffffff24',          // was #ffffff1f — merged with the 0.14 glass edge
  uiInkFaint: '#64607e',
  glassTint: '#0c0a183d',         // = the rgba(12,10,24,0.24) hard-coded in .glass today
  glassFill: '#ffffff0a',         // 0.04
  glassFillHover: '#ffffff14',    // 0.08
  glassHighlight: '#ffffff29',    // 0.16
  glassHighlightStrong: '#ffffff3d', // 0.24
  glassLowlight: '#ffffff0a',     // 0.04
  glassStrokeStrong: '#ffffff38', // 0.22
```

### A.2 `src/theme/applyCssVars.ts` (thêm vào `CSS_VARS`)

```ts
  ['--ink-faint', 'uiInkFaint'],
  ['--glass-tint', 'glassTint'],
  ['--glass-fill', 'glassFill'],
  ['--glass-fill-hover', 'glassFillHover'],
  ['--glass-highlight', 'glassHighlight'],
  ['--glass-highlight-strong', 'glassHighlightStrong'],
  ['--glass-lowlight', 'glassLowlight'],
  ['--glass-stroke-strong', 'glassStrokeStrong'],
```

### A.3 `src/index.css`

```css
:root {
  /* ...các biến màu hiện có (fallback lần vẽ đầu), thêm: */
  --ink-faint: #64607e;
  --glass-tint: #0c0a183d;
  --glass-fill: #ffffff0a;
  --glass-fill-hover: #ffffff14;
  --glass-highlight: #ffffff29;
  --glass-highlight-strong: #ffffff3d;
  --glass-lowlight: #ffffff0a;
  --glass-stroke-strong: #ffffff38;
  --ui-border: #ffffff24;

  /* semantic aliases */
  --glass-stroke: var(--ui-border);
  --glass-saturate: 1.2;
  --glass-edge: inset 0 1px 0 var(--glass-highlight), inset 0 -1px 0 var(--glass-lowlight);
  --glass-edge-strong: inset 0 1px 0 var(--glass-highlight-strong), inset 0 -1px 0 var(--glass-lowlight);
  --shadow-panel: 0 12px 36px -12px rgb(0 0 0 / 0.45);
  --glow-accent: 0 0 14px color-mix(in srgb, var(--accent) 45%, transparent);
  --radius: var(--radius-lg); /* deprecated alias */

  /* motion */
  --dur-press: 120ms;
  --dur-hover: 160ms;
  --dur-enter: 240ms;
  --dur-reveal: 480ms;
  --dur-scene: 1500ms; /* must match CHAT_SLIDE_* in CameraDirector */

  /* Liquid Glass — level 1 defaults (off) */
  --lg-refraction: 0px;
  --lg-bezel: 0px;
  --lg-chroma: 0px;
  --lg-specular: 0;
  --lg-specular-size: 60%;
  --lg-light-x: 0;
  --lg-light-y: -1;
  --lg-dim: 0;
}

@theme {
  --font-sans: "Geist", ui-sans-serif, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  --font-mono: "Geist Mono", ui-monospace, "SF Mono", Menlo, Consolas, monospace;

  /* radius — replaces Tailwind's scale (rounded-xl/2xl/3xl and bare `rounded` stop existing) */
  --radius-*: initial;
  --radius-xs: 4px;
  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-lg: 14px;
  --radius-full: 9999px;

  /* backdrop blur */
  --blur-*: initial;
  --blur-sm: 6px;
  --blur-md: 12px;
  --blur-lg: 16px;

  /* type — text-xs (12), text-sm (14), text-base (16) keep Tailwind defaults */
  --text-micro: 10px;
  --text-micro--line-height: 14px;
  --text-caption: 11px;
  --text-caption--line-height: 16px;
  --text-control: 13px;
  --text-control--line-height: 18px;
  --text-display: 22px;
  --text-display--line-height: 28px;

  /* easing */
  --ease-standard: cubic-bezier(0.2, 0.9, 0.3, 1);
  --ease-spring: cubic-bezier(0.2, 0.9, 0.3, 1.2);
  --ease-scene: cubic-bezier(0.65, 0, 0.35, 1);
}
```

Đã kiểm tra với Tailwind 4.3.3 của dự án:
- `--radius-*: initial` xoá hẳn thang mặc định (`rounded-lg` cũ không còn được sinh ra).
- Biến trong `@theme` sinh ra class tương ứng (`rounded-md`, `backdrop-blur-lg`, `text-caption`…).
- Biến dùng trong CSS thuần bằng `var(--…)` được xuất ra `:root`.

### A.4 `.glass` viết lại theo token (Cấp 1, hình ảnh không đổi)

```css
.glass {
  position: relative;
  isolation: isolate;
  border-radius: var(--radius-lg);
  border: 1px solid var(--glass-stroke);
  box-shadow: var(--shadow-panel), var(--glass-edge);
}
.glass::before {                         /* backdrop layer — Liquid Glass level 3 adds url(#lg-refract) here */
  content: "";
  position: absolute;
  inset: 0;
  z-index: -1;
  border-radius: inherit;
  background: var(--glass-tint);
  backdrop-filter: blur(var(--blur-lg)) saturate(var(--glass-saturate));
  -webkit-backdrop-filter: blur(var(--blur-lg)) saturate(var(--glass-saturate));
}
.glass::after {                          /* specular layer — invisible until --lg-specular > 0 */
  content: "";
  position: absolute;
  inset: 0;
  z-index: 1;
  border-radius: inherit;
  pointer-events: none;
  background: radial-gradient(
    circle at calc(50% + var(--lg-light-x) * 50%) calc(50% + var(--lg-light-y) * 50%),
    rgb(255 255 255 / var(--lg-specular)),
    transparent var(--lg-specular-size)
  );
  mix-blend-mode: plus-lighter;
}
.glass-strong::before { background: var(--ui-surface-strong); }
.glass[data-nested="true"]::before { backdrop-filter: none; -webkit-backdrop-filter: none; }

@media (prefers-reduced-transparency: reduce) {
  .glass::before { background: var(--ui-surface-strong); backdrop-filter: none; -webkit-backdrop-filter: none; }
}
```

> Lưu ý khi chuyển: `backdrop-filter` trên `::before` chỉ lấy mẫu đúng nền khi phần tử cha **không** tạo stacking context có `filter`/`opacity` < 1 ở tổ tiên gần. Kiểm tra lại các animation `fade-in` / `chat-slide-*` (đang dùng `opacity`) bằng ảnh chụp ngay ở bước 4.

---

## Phụ lục B — Bảng quy đổi

### B.1 Bo góc

| Cũ | Vị trí | Mới |
| :-- | :-- | :-- |
| `rounded` (4px) | `HistoryDrawer.tsx` (3), `ChatPanel.tsx:66`, `PerfHUD.tsx` | `rounded-xs` |
| `5px` | `.md code` | `var(--radius-xs)` |
| `6px` | `.score3d`, `.skeleton` | `var(--radius-sm)` (giữ) |
| `8px` | `.cite-tip`, `.md blockquote` (`0 8px 8px 0`) | `var(--radius-sm)` |
| `rounded-lg` (8px) | Dòng `HistoryDrawer.tsx:88`, `CategoryLegend.tsx:42`, nút `AuthOverlay.tsx:144`, `PerfHUD.tsx` | `rounded-md` |
| `10px` | `.btn`, `.field` | `var(--radius-md)` (giữ) |
| `12px` / `rounded-xl` | `.liquid-input-wrap`, `.liquid-cta-btn`; thẻ trong `PaperPanel.tsx:92,100,106`, `ChunkList.tsx:24`, `AuthOverlay.tsx:70` | `rounded-md` / `var(--radius-md)` |
| `!rounded-xl` | Nút mở chat `ChatPanel.tsx:121` | Bỏ `!rounded-xl`, dùng `rounded-lg` của `.glass` (nút kính đứng riêng) |
| `14px` (`--radius`) | `.glass` | `var(--radius-lg)` (giữ) |
| `16px` / `rounded-2xl` | `.liquid-emblem`; bubble `MessageBubble.tsx:46,83` | `rounded-lg` / `var(--radius-lg)` |
| `rounded-br-md`, `rounded-bl-md` (6px) | Đuôi bubble | `rounded-br-sm`, `rounded-bl-sm` (**đổi tên** vì `md` nay là 10px) |
| `999px`, `50%`, `rounded-full` | chip, dot, bar | `var(--radius-full)` / `rounded-full` |

### B.2 Blur

| Cũ | Vị trí | Mới |
| :-- | :-- | :-- |
| `6px` | `.label3d`, `.topk-badge` | `var(--blur-sm)` (giữ) |
| `10px` | `.liquid-badge-chip` (trong cột Chat) | **Bỏ** (kính trong kính) |
| `12px` | `.liquid-emblem`, `.liquid-input-wrap` (trong cột Chat) | **Bỏ** (kính trong kính) |
| `backdrop-blur-md` | `HistoryDrawer.tsx:40` (trong cột Chat) | **Bỏ**; nền `--ui-surface-strong` |
| `14px` | `.liquid-cta-btn` (trong cột Chat) | **Bỏ** (kính trong kính) |
| `16px saturate(1.2)` | `.glass` | `var(--blur-lg)` + `var(--glass-saturate)` (giữ) |

### B.3 Alpha trắng

| Cũ (α) | Mới |
| :-- | :-- |
| 0.03, 0.04, 0.05 | `--glass-fill` (0.04) |
| 0.06, 0.10 | `--glass-fill-hover` (0.08) |
| 0.12 (`--ui-border`), 0.14, 0.15 (viền) | `--glass-stroke` (0.14) |
| 0.14, 0.15, 0.16, 0.18 (highlight `inset`) | `--glass-highlight` (0.16) |
| 0.18 (viền CTA) | `--glass-stroke-strong` (0.22) |
| 0.22, 0.25 (highlight mạnh) | `--glass-highlight-strong` (0.24) |
| 0.04, 0.05, 0.06 (inset dưới) | `--glass-lowlight` (0.04) |
| `#ffffff1f` trong `.topk-badge` | `--glass-highlight` |

### B.4 Cỡ chữ

| Cũ | Vị trí | Mới |
| :-- | :-- | :-- |
| `10px`, `text-[10px]` | `.label3d`, `.topk-meta`, `.digits-ring`, component (5) | `text-micro` / `var(--text-micro)` |
| `11px`, `text-[11px]` | `.label3d` gốc, `.topk-badge`, component (4) | `text-caption` |
| `0.68rem`, `0.7rem`, `0.72rem` | `.liquid-badge-chip`, `.tag`, `.cite-tip`, `.step` | `text-caption` (riêng `.step` và `.cite-tip` dùng `text-xs`) |
| `12px` | `.score3d` | `text-xs` |
| `13px`, `0.8rem`, `0.83rem`, `0.85rem` | `.compare-score`, `.btn`, `.liquid-input`, `.liquid-cta-btn`, `.field` | `text-control` |
| `0.9rem` | `.md` | `text-sm` |
| `22px` | `.query-text` | `text-display` |
| `0.72em`, `0.85em` | `.cite-chip`, `.md code` | Giữ (inline, co theo chữ cha) |
| `text-[0.65rem]` | HealthBanner, TopBar (phụ đề), PipelineStatus (ms), ScoreBars compact, CrawlSettings (tiêu đề nhỏ) | `text-micro` — meta/số liệu phụ *(bổ sung P18, theo vai trò)* |
| `text-[0.68rem]`, `text-[0.7rem]` | CrawlSettings, SettingsPanel (lỗi), MessageBubble (chân), ChunkList (meta), ScoreBars, nút "Bỏ qua" | `text-caption` — nhãn *(bổ sung P18)* |
| `text-[0.78rem]`, `text-[0.8rem]`, `text-[0.82rem]` | CrawlSettings, ResultCard (tiêu đề), ChunkList (đoạn trích), PaperPanel (abstract) | `text-xs` — văn bản đọc trong panel *(bổ sung P18)* |
| `font-weight: 650` | `.md strong` | `600` |

### B.5 Thời gian & easing

| Cũ | Mới |
| :-- | :-- |
| `0.1s`, `0.12s` (transform) | `--dur-press` |
| `0.15s`, `0.16s`, `0.18s`, `0.2s` | `--dur-hover` |
| `0.24s`, `0.25s`, `0.3s`, `0.35s` | `--dur-enter` |
| `0.45s`, `0.5s`, `0.6s` | `--dur-reveal` |
| `1.5s` | `--dur-scene` |
| `cubic-bezier(0.2, 0.8, 0.3, 1)`, `(0.2, 0.9, 0.3, 1)`, `ease-out` cho chuyển trạng thái | `--ease-standard` |
| `cubic-bezier(0.2, 0.9, 0.3, 1.2)` | `--ease-spring` |
| `cubic-bezier(0.65, 0, 0.35, 1)` | `--ease-scene` |

### B.6 Màu

Xem §2.3 (biến không tồn tại) và §2.5 (màu Tailwind mặc định).
