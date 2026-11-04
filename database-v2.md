# database-v2.md — Thiết kế lại Qdrant, MongoDB và MinIO

- **Motivation/Background**: Trước đây Qdrant vừa tìm kiếm vector vừa chứa toàn bộ nội dung chunk. Theo yêu cầu của thầy, nhóm tách lại vai trò và đưa MongoDB lên làm source of truth.
- **Purpose**: Tóm tắt thiết kế mới, điểm khác so với thiết kế cũ và điểm mạnh mang lại.
- **Overview Pipeline**: Dữ liệu đi từ MinIO sang MongoDB để lưu nội dung và sang Qdrant để tìm kiếm; `backend_rag` tìm ở Qdrant rồi đọc nội dung ở MongoDB.
- **Detailed Plan**: §1 Thiết kế mới; §2 Khác gì so với cũ; §3 Điểm mạnh; §4 Trạng thái.
- **References**: [docs/API_DESIGN.md](docs/API_DESIGN.md), [backend_rag/src/vector-store/vector-store.service.ts](backend_rag/src/vector-store/vector-store.service.ts).
- **Created**: 2026-11-05T00:06:57+07:00
- **Last Updated**: 2026-11-05T03:06:00+07:00

[STATUS: IN_PROGRESS]

---

## 1. Thiết kế mới

Mỗi hệ thống chỉ giữ một vai trò:

| Hệ thống | Vai trò | Lưu gì |
| :--- | :--- | :--- |
| **MinIO** | Kho file của pipeline | File gốc (Bronze) và Parquet đã xử lý (Silver) |
| **MongoDB** (`rag_ts_corpus`) | Source of truth | Paper, text của chunk, tác giả, luật kết hợp |
| **Qdrant** | Tìm kiếm vector | Vector và 5 field: `chunk_id`, `paper_id`, `year`, `primary_category`, `section_type` |

```text
MinIO (Parquet) ──sync──→ MongoDB rag_ts_corpus
Câu hỏi → Qdrant trả về chunk_id → MongoDB trả về nội dung → LLM
```

Dữ liệu của ứng dụng như user, token và phiên chat vẫn nằm riêng ở db `rag_ts`.

---

## 2. Khác gì so với cũ

| Hệ thống | Thiết kế cũ | Thiết kế mới |
| :--- | :--- | :--- |
| **Qdrant** | Vector cộng toàn bộ nội dung: `text`, `title`, `authors`, `section_title` | Chỉ còn vector và các field để định danh, lọc |
| **MongoDB** | Chỉ có dữ liệu ứng dụng | Thêm `rag_ts_corpus` chứa nội dung paper và chunk |
| **MinIO** | Chưa có bảng chunk riêng | Thêm `silver/chunks/` dạng Parquet, là nguồn để sync sang MongoDB |
| **Metadata paper** | Lặp lại trên từng chunk | Lưu một lần trong `papers`, chunk tham chiếu qua `paper_id` |

---

## 3. Điểm mạnh

- **Rõ nguồn dữ liệu đúng:** nội dung chỉ có một nơi chuẩn là MongoDB, thay vì nằm trong vector store.
- **Qdrant gọn hơn:** payload không còn chứa text, nên Qdrant chỉ tập trung vào tìm kiếm.
- **Đổi nội dung không phải đụng vector:** sửa metadata hoặc text chỉ cần ghi MongoDB. Chỉ khi text đổi đáng kể mới cần embed lại.
- **Truy vấn linh hoạt hơn:** MongoDB có index và truy vấn theo `paper_id`, tác giả hay danh mục, không cần phụ thuộc vào payload của Qdrant.
- **Kiểm tra được tính toàn vẹn:** `text_sha256`, `content_sha256` và `source_objects` cho phép đối chiếu dữ liệu tới tận file gốc.
- **Khôi phục được:** Parquet trên MinIO là bản sao độc lập, có thể dùng để nạp lại MongoDB.

Đổi lại, mỗi câu hỏi tốn thêm một lượt đọc MongoDB sau khi tìm ở Qdrant. Hiệu năng của luồng mới chưa được đo bằng k6.

---

## 4. Trạng thái

| Phần | Trạng thái |
| :--- | :--- |
| Qdrant | Xong: 1.043.594 point, không còn `text` |
| MinIO | Xong: `silver/chunks/` có đủ 1.043.594 chunk |
| MongoDB | Gần xong với chunk: còn thiếu 827. Còn thiếu ít nhất 11.928/70.638 paper |
| `backend_rag` | Đã có code đọc từ `rag_ts_corpus` (chưa commit), cần đặt `CORPUS_MONGO_URI` trong `.env` |
| Pipeline | Chưa ghi paper mới vào MongoDB |
