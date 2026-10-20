# RUN_GUIDE.md — Chạy RAG_TS (backend + frontend 3D)

MongoDB và Qdrant nằm trên máy chủ, đi qua Cloudflare Tunnel `*.minio-quangmanh.io.vn`. Mỗi người chỉ chạy backend NestJS và frontend Vite trên máy mình.

```
Frontend :5174 ──/api/v1──▶ Backend :3000 ──▶ MongoDB  (mongodb.minio-quangmanh.io.vn, qua cloudflared)
                                         └──▶ Qdrant   (https://qdrant.minio-quangmanh.io.vn, chỉ đọc)
```

## 1. Cần có

- Node.js ≥ 22, npm
- `cloudflared` (`brew install cloudflared` hoặc tải ở trang Cloudflare)
- Repo kèm submodule: `git clone --recurse-submodules <repo RAG_TS>`
- Tài khoản MongoDB (user/password): xin chủ repo, **không commit** vào git

## 2. Mở tunnel MongoDB

MongoDB là TCP nên phải mở một cổng local qua `cloudflared`. Để terminal này chạy suốt lúc làm việc:

```bash
cloudflared access tcp --hostname mongodb.minio-quangmanh.io.vn --url localhost:27018
```

Sau bước này MongoDB dùng được ở `localhost:27018`.

> Dùng 27018 để tránh trùng với MongoDB local (27017) nếu máy bạn có sẵn.

Qdrant đi qua HTTPS nên không cần mở tunnel.

## 3. Backend (`backend_rag`)

```bash
cd backend_rag
cp .env.example .env
npm ci
```

Sửa `.env`:

```dotenv
PORT=3000
CORS_ORIGINS=http://localhost:5174

# MongoDB qua tunnel (bước 2). <user>/<password>: xin chủ repo.
MONGO_URI=mongodb://<user>:<password>@localhost:27018/rag_ts?authSource=admin&directConnection=true

# Qdrant qua tunnel: key chỉ đọc (RAG_TS không bao giờ ghi Qdrant)
QDRANT_URL=https://qdrant.minio-quangmanh.io.vn
QDRANT_API_KEY=uth_guest_readonly_key_4b71a8e9d2c
QDRANT_COLLECTION=scientific_papers_gold

# Mỗi khoá ≥ 32 ký tự, tự đặt
JWT_ACCESS_SECRET=<chuoi-ngau-nhien-it-nhat-32-ky-tu>
JWT_REFRESH_SECRET=<chuoi-ngau-nhien-khac-it-nhat-32-ky-tu>

# LLM: đường dẫn file GGUF trên máy bạn
MODEL_PATH=../model/qwen2.5-7b-instruct-q4_k_m.gguf

# File do pipeline xuất (thiếu thì app vẫn chạy, chỉ tắt tính năng tương ứng)
WORLD_DATA_DIR=../data/world
MINING_DIR=../data/gold/mining
```

Chạy:

```bash
npm run start:dev
```

Kiểm tra: `curl localhost:3000/api/v1/health` trả `"status":"ok"` và `components.mongo`, `components.qdrant`, `components.llm` đều `true`. LLM nạp nền vài giây sau khi khởi động, nên lúc đầu có thể thấy `degraded`. Swagger: http://localhost:3000/api/v1/docs.

## 4. Frontend (`frontend_rag`)

```bash
cd frontend_rag
npm ci
npm run dev:live        # http://localhost:5174
```

`dev:live` gọi backend thật ở `:3000` qua proxy `/api/v1`. World 3D vẫn là mock cho tới khi pipeline xuất dữ liệu world. `npm run dev` chạy toàn mock, không cần backend.

## 5. Thử nhanh

Mở http://localhost:5174, hỏi ví dụ *"How does LoRA reduce the number of trainable parameters?"*. Animation chạy embedding ➔ bay theo cành ➔ so sánh ➔ top-k ➔ câu trả lời có trích dẫn `[Paper: …]`. Mỗi câu mất khoảng 30–40 giây, phần lớn là thời gian LLM sinh câu trả lời.

Chấm **Route** đỏ trên thanh trạng thái là bình thường: pipeline chưa xuất cây world, nên đường bay được dựng tạm từ các paper tìm được.

## 6. Lỗi thường gặp

| Hiện tượng | Nguyên nhân / cách xử lý |
| :-- | :-- |
| Backend dừng ngay, in `Invalid environment` | Thiếu biến trong `.env`; dòng lỗi ghi rõ tên biến |
| `components.mongo: false`, log `Authentication failed` | Sai user/password, hoặc thiếu `authSource=admin` |
| `components.mongo: false`, không kết nối được | Terminal `cloudflared access tcp` (bước 2) chưa chạy hoặc đã tắt |
| `components.qdrant: false` / chat lỗi 503 `qdrant unavailable` | Sai `QDRANT_URL` / `QDRANT_API_KEY`, hoặc máy chủ đang tắt |
| `components.llm: false` / chat lỗi 503 `llm unavailable` | `MODEL_PATH` sai, hoặc model chưa nạp xong (đợi vài giây) |
| Frontend báo "Backend không phản hồi" | Backend chưa chạy ở `:3000`, hoặc đang chạy `npm run dev` thay vì `npm run dev:live` |
| `Port 5174 is already in use` | Tắt tiến trình đang chiếm cổng, hoặc chạy `npm run dev:live -- --port 5175` |

## 7. Ghi chú

- MongoDB trên máy chủ là **standalone** (không replica set). Backend tự chuyển sang ghi tuần tự khi lưu lịch sử chat (log `MONGO_NO_TRANSACTIONS` một lần), không cần cấu hình gì thêm.
- Lịch sử chat chỉ được lưu khi request có JWT. Frontend chưa có màn đăng nhập nên hiện chat ẩn danh; thử luồng có JWT qua Swagger.
- Hợp đồng API: [`API_DESIGN.md`](API_DESIGN.md). Chi tiết backend: [`../backend_rag/README.md`](../backend_rag/README.md).
