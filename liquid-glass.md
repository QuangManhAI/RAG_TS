# liquid-glass.md — Liquid Glass của "Cây Tri Thức Vector"

- **Motivation/Background**: Giao diện nổi trên một cây tri thức 3D, nên kính phải cho thấy cảnh phía sau mà chữ vẫn dễ đọc và 3D vẫn mượt. Đây là một trong những bản cập nhật lớn của frontend.
- **Purpose**: Cung cấp nội dung để làm slide về Liquid Glass đã xây dựng trong `frontend_rag`.
- **Overview Pipeline**: Tổng hợp từ code hiện tại, lịch sử git, báo cáo migration và decision log, rồi sắp xếp theo thứ tự slide.
- **Detailed Plan**: §1 Thông điệp chính; §2 Nội dung từng slide; §3 Lưu ý trước khi trình bày; §4 Quy chuẩn FE chung: màu sắc, bo góc, chữ, chuyển động, trạng thái.
- **References**: [Glass.tsx](frontend_rag/src/components/ui/Glass.tsx), [LiquidShell.tsx](frontend_rag/src/components/ui/LiquidShell.tsx), [LiquidFilters.tsx](frontend_rag/src/components/ui/LiquidFilters.tsx), [refractionMap.ts](frontend_rag/src/theme/refractionMap.ts), [liquidGlass.ts](frontend_rag/src/theme/liquidGlass.ts), [useLiquidLight.ts](frontend_rag/src/theme/useLiquidLight.ts), [design-migration-report.md](frontend_rag/docs/design-migration-report.md), [FE_DESIGN_SYSTEM.md](docs/FE_DESIGN_SYSTEM.md), [tokens.ts](frontend_rag/src/theme/tokens.ts), [Vuong_chat_logs.md](docs/AI_chat_log/Vuong_chat_logs.md), [@samasante/liquid-glass](https://github.com/samasante/liquid-glass).
- **Created**: 2026-11-05T00:22:57+07:00
- **Last Updated**: 2026-11-05T00:26:44+07:00

[STATUS: ACTIVE]

---

## 1. Thông điệp chính

**Biến các panel phẳng thành kính thật: bẻ cong cảnh 3D phía sau, bắt sáng theo con trỏ, và tự hạ cấp theo máy và cài đặt trợ năng của người dùng.**

---

## 2. Nội dung từng slide

### Slide 1 — Vấn đề ban đầu

- Giao diện chạy trên cây 3D với 150.000 điểm vector, nên panel không được che mất cảnh và không được làm chậm cảnh.
- Thứ mang tên `liquid-*` lúc đầu chỉ là glassmorphism thông thường, và chỉ dùng ở form đăng nhập.
- Style bị lệch sau nhiều vòng sửa: 8 mức bo góc, 5 mức blur, 19 cỡ chữ.
- Kính chồng lên kính làm tăng số lớp `backdrop-filter`.

**Ảnh gợi ý:** giao diện cũ đặt cạnh giao diện mới.

### Slide 2 — Liquid Glass khác gì glassmorphism?

| | Glassmorphism | Liquid Glass của dự án |
| :--- | :--- | :--- |
| Nền phía sau | Chỉ làm mờ | Làm mờ và bẻ cong như qua thấu kính |
| Ánh sáng | Viền tĩnh | Viền sáng đổi hướng theo con trỏ |
| Độ dày | Trông như tấm phẳng | Có vát cạnh và ánh sáng tụ trong khối kính |
| Khi máy yếu hoặc cần trợ năng | Thường không có phương án riêng | Tự hạ cấp theo 4 mức |

### Slide 3 — 4 cấp kính đang chạy

| Cấp | Tên trong Cài đặt | Hiệu ứng |
| :--- | :--- | :--- |
| 0 | Đục | Panel đục, không dùng kính |
| 1 | Mờ | Làm mờ nền |
| 2 | | Thêm viền sáng theo con trỏ |
| 3 | Liquid | Thêm khúc xạ nền |

- Cài đặt có 4 lựa chọn: Tự động, Đục, Mờ, Liquid.
- **Cấp 4** dùng kính WebGL mới nằm trong thiết kế, chưa làm.

**Ảnh gợi ý:** cùng một panel trên nền sọc, so sánh `glass-backdrop-frosted` và `glass-backdrop-liquid` trong [ảnh mốc visual test](frontend_rag/e2e/visual.spec.ts-snapshots/).

### Slide 4 — Kiến trúc

```text
Glass              Một component kính duy nhất, cho mọi bề mặt
├── ::before       Lớp màu nền
├── ::after        Viền sáng 1,5 px theo con trỏ
└── Token CSS      Đổi cấp bằng token, không phải sửa component

LiquidShell        Bọc panel bằng thấu kính khi ở cấp 3
LiquidFilters      Tạo bản đồ ánh sáng vát cạnh theo kích thước panel
useLiquidLight     Tính hướng sáng theo con trỏ
```

- 4 panel dùng thấu kính ở cấp 3: khung chat, thanh trên cùng, bảng "Loại bài báo" và bảng Cài đặt.
- Phần khúc xạ cấp 3 dùng thư viện mã nguồn mở `@samasante/liquid-glass` (MIT). Thông số quang học được tinh chỉnh tay cho từng panel.

### Slide 5 — Phần toán đáng nói

- **Hình dạng kính:** dùng hàm khoảng cách có dấu (SDF) của hình chữ nhật bo góc để biết mỗi điểm cách mép bao xa và mép quay về hướng nào.
- **Bản đồ dịch chuyển:** kênh R và G của ảnh lưu độ lệch theo x và y, với 128 là không lệch. Bộ lọc SVG `feDisplacementMap` dùng ảnh này để bẻ cong hình.
- **Chống gập ảnh:** giới hạn độ dốc tổng của thấu kính dưới 0,92. Nếu dốc tới 1, nền sẽ bị gập ngược và tạo một đường cứng song song với mép.
- **Hướng sáng bằng số phức:** biểu diễn hướng bằng vector đơn vị (rotor) và nội suy `nlerp`, không dùng góc hay `atan2`.

```text
Hướng sáng = (dx, dy) / √(dx² + dy²)      (dx, dy: từ tâm màn hình tới con trỏ)
```

### Slide 6 — Những lỗi khó đã giải quyết

| Lỗi | Nguyên nhân | Cách sửa |
| :--- | :--- | :--- |
| Một mảng sáng lớn phủ lên thanh trên cùng | Ánh sáng phủ cả bề mặt | Chỉ vẽ ở viền 1,5 px |
| Panel phía sau lộ rõ qua bảng Cài đặt | `mix-blend-mode` làm hỏng blur trên Chromium | Bỏ `mix-blend-mode` |
| Mất blur phần lớn số lần tải | Đặt blur ở `::before` | Đặt blur trực tiếp trên phần tử |
| Rê chuột làm chậm khung hình 11% | Ghi hướng sáng lên `<html>` mỗi khung hình | Ghi riêng cho từng bề mặt và tách viền thành layer riêng |
| Panel nhấp nháy khi xoay cảnh | Cấp kính phụ thuộc cấp chất lượng 3D luôn thay đổi | Tách cấp kính khỏi cấp 3D |
| Chữ khó đọc trên cành cây sáng | Thấu kính để lộ các cành cây sáng phía sau | Chữ trắng 3 mức, kèm viền tối mỏng quanh chữ |

**Số liệu cho lỗi blur:** đo độ lệch chuẩn độ sáng trên nền sọc, khoảng 4 là mờ đúng và 80 là sọc vẫn rõ. Trước khi sửa, kết quả ở cấp 1 là 80 · 3,7 · 80 · 80 · 80. Sau khi sửa là 3,7 ở cả 5 lần.

### Slide 7 — Hiệu năng

Thời gian mỗi khung hình khi rê chuột, với 150.000 vector:

| Cấu hình | Median | p95 |
| :--- | :--- | :--- |
| Cấp 1 | 2,5–2,7 ms | 3,6–4,0 ms |
| Cấp 2 | 2,4–2,5 ms | 4,1–4,9 ms |
| Cấp 3, tách màu 0,5 px | 4,3–4,6 ms | 5,6–6,7 ms |
| **Cấp 3, không tách màu** | **2,5 ms** | **4,4–5,0 ms** |

- Bản đo này khiến nhóm bỏ tách màu ở bản khúc xạ tự viết, vì nó làm median tăng gần gấp đôi.
- FCP và LCP gần như không đổi: cấp 1 là 232 ms và 248 ms, cấp 3 là 244 ms và 244 ms.

> **Bắt buộc ghi kèm:** số đo ngày 2026-10-30, chạy headless không vsync, trên bản khúc xạ tự viết. Bản dùng thư viện hiện tại chưa được đo (§3).

### Slide 8 — Trợ năng và tự hạ cấp

| Điều kiện | Cấp tối đa |
| :--- | :--- |
| Người dùng chọn Đục, hoặc hệ điều hành bật giảm trong suốt | 0 |
| Người dùng chọn Mờ | 1 |
| Hệ điều hành bật tăng tương phản | 2 |
| Chất lượng đặt ở mức Thấp | 2 |
| Không phải Chromium | 2 |
| Chromium, chọn Tự động hoặc Liquid | 3 |

- Khi bật giảm chuyển động, viền sáng đứng yên, không chạy theo con trỏ.
- Hệ điều hành đổi cài đặt trợ năng thì cấp kính được tính lại ngay.

### Slide 9 — Kết quả

| Chỉ số | Trước | Sau |
| :--- | :--- | :--- |
| Mức bo góc (không tính `full`) | 8 | 4 |
| Mức blur (không tính 0 px) | 5 | 3 |
| Mức alpha trắng | 12 | 6 |
| Cỡ chữ | 19 | 9 |
| Lớp `backdrop-filter` ở màn đăng nhập | 7 | 3 |
| Lớp `backdrop-filter` ở màn lịch sử | 5 | 3 |
| Lớp `backdrop-filter` ở màn trả lời | 10 | 9 |

- Một component `Glass` thay cho các style kính rời rạc.
- Quy tắc **không chồng kính lên kính** được đưa vào design system.
- Có bộ kiểm tra `check:design` và ảnh mốc `test:visual` để giữ giao diện không bị lệch lại.

Các số lớp `backdrop-filter` được đo ở bước tạo component `Glass`, trước khi có cấp 2 và cấp 3.

### Slide 10 — Dòng thời gian

| Ngày | Mốc |
| :--- | :--- |
| 2026-10-21 đến 10-22 | Form đăng nhập kiểu glassmorphism đầu tiên |
| 2026-10-30 | Thêm token, component `Glass`, cấp 2 (viền sáng) và cấp 3 (khúc xạ SVG tự viết) |
| 2026-11-03 | Tách cấp kính khỏi cấp 3D, tích hợp thư viện thấu kính, áp dụng cho 4 panel và tinh chỉnh quang học |

### Slide 11 — Hướng tiếp theo

- Đo lại hiệu năng trên bản dùng thư viện và trên GPU thật.
- Mở cấp 3 cho Safari và Firefox. Thư viện có hỗ trợ, nhưng code hiện chỉ cho Chromium lên cấp 3.
- Làm cấp 4 bằng kính WebGL vẽ trực tiếp trong cảnh 3D.

---

## 3. Lưu ý trước khi trình bày

**Nên nói rõ:**

1. **Có dùng thư viện:** phần khúc xạ cấp 3 hiện do `@samasante/liquid-glass` 0.1.1 thực hiện. Phần do dự án tự làm gồm hệ thống cấp kính, component `Glass`, ánh sáng vát cạnh, viền sáng theo con trỏ, thông số quang học và phần trợ năng. Bản khúc xạ SVG tự viết vẫn còn trong code, nhưng không còn là lớp bẻ cong nền trên 4 panel này.
2. **Lấy cảm hứng từ Apple:** đây là bản dựng lại trên web theo ý tưởng Liquid Glass của Apple, không phải công nghệ của Apple.

**Chưa nên khẳng định:**

1. **Chưa có số đo cho bản hiện tại.** Bảng ở Slide 7 có trước khi tích hợp thư viện. Đừng nói "kính hiện tại gần như không tốn hiệu năng" khi chưa đo lại, vì:
   - Thiết kế ban đầu đặt ngân sách tối đa 1 bề mặt khúc xạ lớn, còn hiện có 4 panel.
   - Thư viện đang đặt `dispersion: 1`, trong khi bản đo cũ cho thấy tách màu rất tốn.
2. **Mức 5,7 FPS trong ảnh chạy tự động chưa kết luận được gì.** Lần chạy đó có thể không dùng GPU thật. Hãy chạy `npm run test:perf` và xem HUD trên máy demo trước khi đưa số lên slide.

**Khi chụp ảnh cho slide:**

- Chụp trên máy thật khi cây 3D đã hiện đầy đủ, vì trên nền đen kính gần như không thấy.
- Đừng dùng ảnh trả lời hiện tại làm minh hoạ, vì chat chưa có dữ liệu thật và đang trả lời "không đủ thông tin".
- Mở URL không có `?perf` để ẩn HUD FPS, vì HUD che bảng "Loại bài báo".
- Nên có một đoạn video ngắn rê chuột qua panel, vì viền sáng và khúc xạ thấy rõ nhất khi chuyển động.

---

## 4. Quy chuẩn FE chung

Nguồn chuẩn là [docs/FE_DESIGN_SYSTEM.md](docs/FE_DESIGN_SYSTEM.md). Giá trị dưới đây đã đối chiếu với [tokens.ts](frontend_rag/src/theme/tokens.ts) và [index.css](frontend_rag/src/index.css). Lần chạy `npm run check:design` ngày 2026-11-05 cho 0 vi phạm ở cả 14 quy tắc.

### 4.1 Quy tắc vàng

1. **Không dùng giá trị tự do trong component.** Màu, bo góc, blur, alpha, cỡ chữ, bóng và thời gian chuyển động đều lấy từ token.
2. **Cấm** hex và `rgba()` viết thẳng trong `.tsx`, giá trị dạng `rounded-[12px]`, và màu Tailwind như `red-500` hay `text-white`.
3. **Chỉ dùng CSS variable đã khai báo.** Không dùng fallback để phòng hờ biến không tồn tại.
4. **Không đặt kính lên kính.** Phần tử bên trong panel kính chỉ dùng lớp tô, không thêm `backdrop-filter`.
5. **Bo góc đồng tâm.** Bo góc trong = bo góc ngoài − padding, rồi làm tròn về bậc gần nhất.
6. **Màu nhấn chỉ dành cho hành động chính.**
7. **Màu category là dữ liệu,** không dùng để trang trí.

### 4.2 Màu sắc

| Vai trò | CSS var | Giá trị | Dùng cho |
| :--- | :--- | :--- | :--- |
| Nền trang | `--bg` | `#070614` | Nền body, phía sau cảnh 3D |
| Chữ chính | `--ink` | `#ece9ff` | Nội dung đọc |
| Chữ phụ | `--ink-muted` | `#a29cc4` | Mô tả, nhãn, meta |
| Chữ mờ | `--ink-faint` | `#64607e` | Chỉ cho placeholder, timestamp, disabled |
| Nhấn | `--accent` | `#7fe8ff` | Hành động chính, focus, liên kết, trạng thái active |
| Chữ trên nền nhấn | `--accent-ink` | `#06121a` | Chữ và icon trên `--accent` |
| Cảnh báo | `--warn` | `#ffc145` | Chưa xác minh, offline, lưu ý |
| Nguy hiểm | `--danger` | `#ff5d73` | Lỗi, xoá |
| Thành công | `--success` | `#58f0a7` | Hoàn tất |
| Bề mặt vừa | `--ui-surface` | `#120f2499` | Nhãn 3D, nền nút |
| Bề mặt đặc | `--ui-surface-strong` | `#151129e6` | Panel cần đọc lâu |

**Màu kính:**

| CSS var | Giá trị | Dùng cho |
| :--- | :--- | :--- |
| `--glass-tint` | `#0c0a183d` (α 0,24) | Lớp màu nền của kính |
| `--glass-fill` | `#ffffff0a` (α 0,04) | Control bên trong kính, lúc nghỉ |
| `--glass-fill-hover` | `#ffffff14` (α 0,08) | Control khi hover hoặc focus |
| `--ui-border` | `#ffffff24` (α 0,14) | Viền 1 px của kính và control |
| `--glass-stroke-strong` | `#ffffff38` (α 0,22) | Viền khi hover, hoặc khi bật tăng tương phản |
| `--glass-highlight` | `#ffffff29` (α 0,16) | Viền sáng phía trên |
| `--glass-highlight-strong` | `#ffffff3d` (α 0,24) | Viền sáng của CTA và focus |
| `--glass-lowlight` | `#ffffff0a` (α 0,04) | Viền sáng phía dưới |

**Màu category:**

| Category | Màu | Category | Màu |
| :--- | :--- | :--- | :--- |
| `cs.LG` | `#38d6ff` | `cs.IR` | `#ff7a45` |
| `cs.CL` | `#ff3d9a` | `cs.RO` | `#2ff5c8` |
| `cs.CV` | `#8dff4a` | `math.OC` | `#ff4d5e` |
| `cs.AI` | `#ffc23d` | `q-bio.QM` | `#4d7cff` |
| `stat.ML` | `#a98bff` | `physics.comp-ph` | `#f08cff` |

Bảng còn 2 màu dự phòng (`#e8ff5a`, `#d9e6ff`) và màu `#8a7fa8` cho category khác.

**Tỷ lệ pha màu với `color-mix`:**

| Mục đích | Tỷ lệ |
| :--- | :--- |
| Nền rất nhẹ | 6% |
| Nền nhẹ: chip, hover | 12% |
| Nền vừa: active, bubble người dùng | 16% |
| Nền đậm: highlight | 24% |
| Viền màu | 40% |
| Viền màu khi hover hoặc focus | 55% |
| Glow | 45% hoặc 70% |

**Độ tương phản trên `--bg`,** được kiểm tra bằng [contrast.test.ts](frontend_rag/src/theme/contrast.test.ts):

| Cặp màu | Yêu cầu |
| :--- | :--- |
| `--ink` | Từ 7:1 |
| `--ink-muted` | Từ 4,5:1 |
| `--ink-faint` | Từ 3:1 và dưới 4,5:1, nên không dùng cho chữ cần đọc |
| `--accent-ink` trên `--accent` | Từ 4,5:1 |

### 4.3 Bo góc, blur, chữ

| Bo góc | Giá trị | Dùng cho |
| :--- | :--- | :--- |
| `xs` | 4 px | `code` inline, badge rất nhỏ |
| `sm` | 6 px | Tooltip, skeleton |
| `md` | 10 px | Nút, ô nhập, dòng danh sách, thẻ trong panel |
| `lg` | 14 px | Panel kính, bubble chat |
| `full` | 9999 px | Chip, tag, chấm tròn |

| Blur | Giá trị | Dùng cho |
| :--- | :--- | :--- |
| `sm` | 6 px | Nhãn nhỏ trong 3D |
| `md` | 12 px | Control đứng riêng trên cảnh 3D |
| `lg` | 16 px | Panel |

| Cỡ chữ | Cỡ / dòng | Dùng cho |
| :--- | :--- | :--- |
| `text-micro` | 10 / 14 | Meta phụ |
| `text-caption` | 11 / 16 | Chip, tag, nhãn 3D |
| `text-xs` | 12 / 16 | UI dày đặc: danh sách, tooltip |
| `text-control` | 13 / 18 | Nút, ô nhập |
| `text-sm` | 14 / 20 | Nội dung chat |
| `text-base` | 16 / 24 | Tiêu đề panel |
| `text-display` | 22 / 28 | Chữ câu hỏi bay trong 3D |

- **Font:** Geist cho giao diện, Geist Mono kèm `tabular-nums` cho số đo như điểm cosine, ms, FPS.
- **Độ đậm:** 400, 500 cho nhãn và nút, 600 cho tiêu đề và CTA, 700 cho điểm số nổi bật.
- **Khoảng cách:** theo lưới 2 px và 4 px, từ 2 px đến 32 px.
- **Icon:** cỡ icon bằng cỡ chữ đi kèm cộng 2 px, gồm các bậc 12, 14, 16 và 28.

### 4.4 Chuyển động

| Token | Giá trị | Dùng cho |
| :--- | :--- | :--- |
| `--dur-press` | 120 ms | Khi nhấn |
| `--dur-hover` | 160 ms | Đổi màu khi hover hoặc focus |
| `--dur-enter` | 240 ms | Xuất hiện và biến mất |
| `--dur-reveal` | 480 ms | Thẻ bật ra, intro |
| `--dur-scene` | 1500 ms | Khung chat trượt cùng camera |

| Easing | Giá trị | Dùng cho |
| :--- | :--- | :--- |
| `--ease-standard` | `cubic-bezier(0.2, 0.9, 0.3, 1)` | Mặc định |
| `--ease-spring` | `cubic-bezier(0.2, 0.9, 0.3, 1.2)` | Phần tử bật ra, có vượt nhẹ |
| `--ease-scene` | `cubic-bezier(0.65, 0, 0.35, 1)` | Chuyển động lớn cùng camera |

### 4.5 Trạng thái tương tác

| Trạng thái | Quy tắc |
| :--- | :--- |
| Hover | Nền lên một bậc, viền accent 55% |
| Nhấn | `translateY(1px)` |
| Đang bật | Viền và chữ `--accent`, nền accent 16% |
| Focus bàn phím | `outline: 2px solid var(--accent)`, `outline-offset: 2px` |
| Disabled | `opacity: 0.45`, `cursor: not-allowed` |
| Lỗi | Viền `--danger` |

### 4.6 Ngoại lệ ở cấp 3

Trên thấu kính, nền phía sau sáng và thay đổi liên tục, nên panel dùng bộ chữ riêng:

| Token | Giá trị ở cấp 3 |
| :--- | :--- |
| `--ink` | Trắng |
| `--ink-muted` | Trắng 72% |
| `--ink-faint` | Trắng 48% |

Chữ có thêm viền tối mỏng để đọc được trên các cành cây sáng. Nút đang chọn dùng chữ trắng trên nền accent đặc.

### 4.7 Slide gợi ý: Quy chuẩn thiết kế

- **Một nguồn token duy nhất,** dùng chung cho giao diện và cảnh 3D.
- **Thang giá trị gọn:** 4 mức bo góc, 3 mức blur, 6 mức alpha trắng, 9 cỡ chữ.
- **Có kiểm soát tự động:** `check:design` với 14 quy tắc, test độ tương phản và ảnh mốc `test:visual`.
