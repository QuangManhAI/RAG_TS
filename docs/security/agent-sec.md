# agent-sec.md — Báo cáo bảo mật backend_rag

- **Motivation/Background**: Trước khi đưa `RAG_TS` lên production, `backend_rag` được review bảo mật và sửa lỗi. Báo cáo này lưu lại kết quả để nhóm theo dõi các việc còn mở.
- **Purpose**: Ghi lại các lỗi đã tìm thấy, cách sửa, bằng chứng kiểm tra và những việc chủ dự án còn phải làm.
- **Overview Pipeline**: Cài skill `cloudflare/security-audit-skill` → review có trọng tâm do một agent thực hiện → sửa code và cấu hình → kiểm tra bằng test và chạy BE thật.
- **Detailed Plan**: §1 Kết luận và danh sách lỗi theo mức độ; §2 Việc phải làm; §3 Phạm vi; §4 Chi tiết từng lỗi; §5 Kiểm tra; §6 Không thấy lỗi; §7 Chưa kiểm tra và giới hạn.
- **References**: [AGENTS.md](../../AGENTS.md), [docs/prompts/prompt_security.md](../prompts/prompt_security.md), [docs/AI_chat_log/Vuong_chat_logs.md](../AI_chat_log/Vuong_chat_logs.md) (D-054 đến D-057), [backend_rag/docker-compose.yml](../../backend_rag/docker-compose.yml), [.claude/skills/security-audit/SKILL.md](../../.claude/skills/security-audit/SKILL.md).
- **Created**: 2026-11-05T02:11:58+07:00
- **Last Updated**: 2026-11-06T13:13:54+07:00

[STATUS: IN_PROGRESS]

> **Không commit file này lên repo public trước khi hoàn tất §2.1 và §2.2.** Báo cáo không chứa secret, nhưng cho biết các điểm yếu chưa được đóng.

---

## 1. Kết luận

**Đã sửa toàn bộ lỗi nằm trong `backend_rag`, nhưng hệ thống chưa an toàn hẳn.** Hai credential đã lộ trên GitHub public vẫn chưa được đổi, và các dịch vụ của pipeline vẫn mở trên mọi network interface.

Có **10 lỗi**, xếp từ nặng đến nhẹ. Trong cùng một mức, lỗi còn mở hoặc ảnh hưởng rộng hơn đứng trước.

| # | Mức | Lỗi | Hậu quả chính | Trạng thái |
| :--- | :--- | :--- | :--- | :--- |
| 1 | **Critical** | Mongo của app không có mật khẩu, mở ra mọi interface | Đọc được toàn bộ user và chat, tự nâng quyền admin | Đã sửa |
| 2 | **Critical** (nếu mật khẩu còn dùng được) | Mật khẩu `app_user` lộ trên GitHub public | Đọc và ghi mọi db trên `mongo_service`, kể cả `rag_ts_corpus` | **Còn mở: phải đổi mật khẩu** |
| 3 | **Medium** | Key read-only của Qdrant lộ trên GitHub public | Đọc được vector và metadata, không ghi được | **Còn mở: phải đổi key** |
| 4 | **Medium** | Giả mạo IP bằng `X-Forwarded-For` | Vượt rate limit và khóa đăng nhập, làm giả IP trong audit log | Đã sửa |
| 5 | **Medium** | Một client chiếm hết hàng đợi LLM | Người dùng khác nhận 429 | Đã sửa |
| 6 | **Medium** | Thư viện có lỗ hổng: 11 lỗi production theo `npm audit` | Tùy thư viện, chưa chứng minh khai thác được | Đã sửa |
| 7 | **Low** | Service admin không gọi `assertAdmin` | Mất lớp phòng thủ thứ hai, chưa bị bypass vì controller có guard | Đã sửa |
| 8 | **Low** | Cache `token_version` được gia hạn liên tục | Trạng thái cũ có thể kéo dài tới 15 phút | Đã sửa |
| 9 | **Low** | Refresh thiếu kiểm tra `is_active` ở nhánh có cache | Tài khoản vừa bị khóa có thể refresh trong một race hẹp | Đã sửa |
| 10 | **Low** | `actor_email` trong audit log luôn là `unknown` | Audit log thiếu email, `actor_id` vẫn đúng | Đã sửa |

**Cách xếp mức:** agent tự đánh giá theo thang của skill `security-audit`, dựa trên hậu quả có thể xảy ra. Lỗi 6 được xếp Medium dù `npm audit` có 2 critical và 5 high, vì mức của `npm audit` không xét việc lỗ hổng có khai thác được trong app này hay không. Việc đó chưa được kiểm chứng.

Toàn bộ thay đổi vẫn nằm trong working tree, **chưa commit**.

---

## 2. Việc phải làm

Trạng thái bên dưới được kiểm tra lúc 2026-11-05 02:11 (+07:00). Các việc này cần quyền sudo hoặc ảnh hưởng đến đồng đội, nên agent không tự làm.

### Bắt buộc trước khi bật lại tunnel

1. **Đổi mật khẩu `app_user`, hoặc xóa user nếu không ai dùng.** Chạy trong `mongosh` của `mongo_service`:
   ```js
   db.getSiblingDB("admin").changeUserPassword("app_user", "<mật khẩu mới>")
   ```
2. **Đổi key read-only của Qdrant.** Sửa `QDRANT__SERVICE__READ_ONLY_API_KEY` trong cấu hình ở `~/Project/docker`, tạo lại container, rồi cập nhật `backend_rag/.env`. Cần báo đồng đội và pipeline trước.
3. **Bật firewall.** Hiện vẫn đang tắt:
   ```bash
   sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
   ```

### Nên làm tiếp

4. **Đóng cổng của pipeline.** `mongo_service` (27017), `qdrant_service` (6333), `minio_service` (9000–9001) và `rabbitmq_service` (5672, 15672) vẫn mở `0.0.0.0`.
5. **Khởi động lại các BE đang chạy nếu chúng dùng Mongo 27018.** Các tiến trình ở cổng 3000 và 3001 không do agent quản lý nên chưa được đụng tới. Chúng cần `.env` mới để đăng nhập được vào Mongo.
6. **Xóa project Mongo tạm.** Volume của nó vẫn còn. Chạy trong `backend_rag/`:
   ```bash
   docker compose -p ragts-sectest down -v
   ```
7. **Đặt `TRUST_PROXY_HOPS`** đúng số proxy thật khi bật tunnel hoặc proxy. Với số proxy sai, rate limit có thể tính sai IP.
8. **Gửi credential Mongo mới cho đồng đội** nếu họ dùng instance `backend_rag-mongo-1`.

---

## 3. Phạm vi

- **Đã review:** auth và token, phân quyền admin, tách dữ liệu giữa các user, prompt injection ở phần RAG, rate limit, cấu hình Mongo và Docker, secret trong git, thư viện.
- **Cách làm:** review có trọng tâm theo checklist của skill, không chạy đủ 6 phase. Một agent vừa tìm vừa kiểm tra, nên **chưa có kiểm chứng độc lập cho từng phát hiện** và đây không phải audit đầy đủ.
- **Ngoài phạm vi:** `frontend_rag` và stack pipeline trong `~/Project/docker`.

---

## 4. Chi tiết từng lỗi

Số mục khớp với số thứ tự ở bảng §1.

### 4.1 Critical: Mongo của app không có mật khẩu

- **Lỗi:** `backend_rag-mongo-1` không bật auth và mở `0.0.0.0:27018`, trong khi firewall đang tắt.
- **Hậu quả:** ai vào được cổng này sẽ đọc được user, hash mật khẩu và toàn bộ chat. Họ cũng tự nâng quyền admin được bằng cách sửa `roles`, vì guard đọc role từ DB.
- **Cách sửa (D-056):** bật auth bằng `keyFile`, chỉ mở `127.0.0.1`, và cho app dùng tài khoản `ragts_app` chỉ có quyền `readWrite` trên `rag_ts`.
- **Kết quả:** kết nối ẩn danh bị từ chối, cổng đóng trên các IP không phải loopback, volume giữ nguyên và số document không đổi.

### 4.2 Critical nếu mật khẩu còn dùng được: mật khẩu `app_user` bị lộ

- **Lỗi:** mật khẩu của `app_user` nằm trong lịch sử GitHub public (3 commit ở `RAG_TS`, 2 commit ở `backend_rag`). Tài khoản này có quyền `readWriteAnyDatabase` trên `mongo_service`, là instance chứa `rag_ts_corpus`.
- **Hậu quả nếu mật khẩu còn dùng được:** kẻ tấn công vào được cổng 27017 có thể đọc và ghi mọi db, kể cả source of truth. Khi đó mức độ là **Critical**.
- **Chưa kiểm tra:** agent không thử đăng nhập bằng mật khẩu này. Hãy coi như nó vẫn dùng được.
- **Đã làm:** thay bằng placeholder trong `README.md`, `docs/guides/RUN_GUIDE.md` và `docs/prompts/prompt_security.md`.
- **Còn mở:** đổi mật khẩu theo §2.1. Lịch sử git không được viết lại.
- **Hai sự cố trong quá trình review:**
  - Báo cáo review đầu tiên sai khi xem mật khẩu này chỉ còn trong lịch sử git. Thực tế lúc đó nó vẫn nằm trong file hiện tại.
  - Một lệnh đọc `README.md` đã in mật khẩu vào log của phiên làm việc. Đây thêm một lý do để đổi mật khẩu.

### 4.3 Medium: API key Qdrant bị lộ trên repo public

- **Lỗi:** key read-only đang dùng từng nằm trong `README.md` và `docs/guides/RUN_GUIDE.md`, có trên `origin/main` từ 2026-10-20. Qdrant mở `0.0.0.0:6333`.
- **Hậu quả:** key chỉ đọc được vector và metadata, không ghi được.
- **Đã làm:** thay bằng placeholder trong file hiện tại.
- **Còn mở:** đổi key theo §2.2, vì key vẫn còn trong lịch sử git.

### 4.4 Medium: Giả mạo IP để vượt rate limit

- **Lỗi:** ở production, `trust proxy` luôn bằng 1, trong khi compose mở thẳng cổng app mà không có proxy.
- **Hậu quả:** gửi `X-Forwarded-For` giả sẽ vượt được rate limit và khóa đăng nhập theo `email|ip`, đồng thời làm giả IP trong audit log.
- **Cách sửa (D-054):** thêm `TRUST_PROXY_HOPS`, mặc định 0, không suy ra từ `NODE_ENV`. Cổng app mặc định chỉ mở `127.0.0.1`.
- **Kết quả:** trên BE thật, đổi `X-Forwarded-For` ở mỗi lần đăng nhập sai vẫn nhận `401` năm lần rồi `429`.

### 4.5 Medium: Một client chiếm hết hàng đợi LLM

- **Lỗi:** LLM chỉ có 1 slot và 8 chỗ chờ, mỗi lượt tối đa 60 giây, trong khi mỗi IP được gọi 20 lần/phút mà không cần đăng nhập.
- **Hậu quả:** một client ẩn danh có thể làm người dùng khác nhận 429.
- **Cách sửa (D-055):** tối đa 2 request đang chạy mỗi user, 3 mỗi IP. Ẩn danh còn 10 lần/phút, có header `Authorization` là 20 lần/phút.
- **Giới hạn:** không chống được tấn công từ nhiều IP.

### 4.6 Medium: Thư viện có lỗ hổng

- **Lỗi:** `npm audit` báo 11 lỗi production (2 critical, 5 high), trong đó có `body-parser` xử lý mọi request JSON và `simple-git` đến từ `node-llama-cpp`.
- **Cách sửa (D-057):** chạy `npm audit fix` và override `simple-git`, `multer`, `undici`.
- **Kết quả:** `npm audit --omit=dev` còn 0 lỗi.

### 4.7 đến 4.10 Low

| # | Lỗi | Hậu quả | Cách sửa |
| :--- | :--- | :--- | :--- |
| 4.7 | `assertAdmin` được định nghĩa nhưng không nơi nào gọi, trái invariant 8 của `AGENTS.md` | Mất lớp phòng thủ thứ hai, chưa bị bypass vì controller có guard | Thêm `src/admin/assert-admin.ts`, gọi ở 13 entry point |
| 4.8 | `updateAgeOnGet` gia hạn liên tục cache `token_version` có TTL 5 giây | Sửa DB trực tiếp hoặc chạy nhiều instance thì trạng thái cũ có thể kéo dài đến khi access token hết hạn (15 phút) | Bỏ `updateAgeOnGet` |
| 4.9 | Nhánh có cache của refresh không kiểm tra `is_active` | Tài khoản vừa bị khóa có thể refresh được trong một race hẹp | Kiểm tra `!live \|\| !live.is_active` |
| 4.10 | Guard không gắn email vào `req.user` | `actor_email` trong audit log luôn là `unknown` (`actor_id` vẫn đúng) | Guard gắn `{ id, roles, email }` |

---

## 5. Kiểm tra

| Hạng mục | Kết quả |
| :--- | :--- |
| `npx jest --runInBand` | 19 suite, **228/228 pass** |
| `npm run build` | Thành công |
| `npm audit --omit=dev` | **0 lỗi** |
| `docker compose config --quiet` | Hợp lệ |
| Cài mới bằng compose | Ẩn danh bị từ chối; `ragts_app` chạy được transaction; ghi vào `other` và `admin` bị từ chối |
| Số document Mongo trước và sau khi bật auth | Không đổi: `chat_messages` 34, `chat_sessions` 14, `refresh_tokens` 11, `users` 1 |
| Cổng 27018 trên IP không phải loopback | Đóng |
| BE thật với `.env` mới | Health tốt với Mongo, Qdrant, LLM; không có lỗi auth Mongo trong log; chat ẩn danh trả 200 |
| Quét file đang được git theo dõi (`RAG_TS`, `backend_rag`, `frontend_rag`) | Không còn key Qdrant đang dùng, không còn URI Mongo của `app_user` có mật khẩu |

---

## 6. Không thấy lỗi

Các kết luận này đến từ đọc source, không phải kiểm thử xâm nhập.

- **Tách dữ liệu giữa các user:** session, message và idempotency key đều lọc theo chủ sở hữu.
- **NoSQL injection:** filter và sort theo whitelist, regex được escape.
- **Token:** role lấy từ DB chứ không tin JWT; refresh token xoay vòng nguyên tử.
- **XML:** chặn DOCTYPE và entity.
- **Prompt injection:** LLM không có tool nên chưa vượt được ranh giới nào ở BE. Cách FE render câu trả lời thì chưa được kiểm tra.

---

## 7. Chưa kiểm tra và giới hạn

- **Code chưa đọc hết:** `tree-router`, `mining`, một phần `admin.dto` và `vector-store`.
- **Frontend:** chưa review, và chưa chạy lại với BE mới.
- **Tunnel:** không đọc được cấu hình route của Cloudflare. Nếu tunnel trỏ vào BE ở chế độ `development`, Swagger sẽ lộ ra ngoài.
- **Credential của MinIO và RabbitMQ:** chưa kiểm tra.
- **`test:e2e`:** chưa chạy lại sau các bản sửa.
- **`TRUST_PROXY_HOPS` lớn hơn 0:** chỉ có unit test. Bài chạy BE thật dùng giá trị 0.
- **`simple-git` 4:** là bản tăng major so với mức `node-llama-cpp` yêu cầu. Mới kiểm tra import, chưa chạy đường build llama.cpp từ source.
- **`/health`:** kiểm tra Mongo bằng `ping`, lệnh này chạy được khi chưa đăng nhập, nên không phát hiện được credential sai.
- **Nâng cấp từ dữ liệu cũ:** không thử trước trên project tạm mà làm thẳng trên bản thật, rồi đối chiếu số document.
- **k6:** kịch bản chat từ một IP sẽ gặp 429 khi có hơn 3 request đang chạy, hoặc vượt 10/20 lần mỗi phút. Cần tính đến khi thiết kế bài test.
