# lesson_2.md — Course Notes on AI-Assisted Software Engineering Methodology (Archived)

- **Motivation/Background**: Lecture notes from Lesson 2 establishing the Requirement -> Specification -> Pseudocode -> Structured Prompt -> Verification workflow used during early project bootstrapping.
- **Purpose**: Historical pedagogical reference for human-AI pair programming principles. Superseded for active repository engineering by AGENTS.md and docs/INDEX.md.
- **Overview Pipeline**: Requirement Clarification -> Input/Output Specification -> Pseudocode Contract -> Structured AI Prompt -> Test & Trace Verification -> Human Decision Log.
- **Detailed Plan**: Preserves original Vietnamese lecture notes on developer roles, AI safety rules, specification tables, and verification loops.
- **References**: [../AGENTS.md](../AGENTS.md), [INDEX.md](INDEX.md), [AI_chat_log/Vuong_chat_logs.md](AI_chat_log/Vuong_chat_logs.md).
- **Created**: 2026-10-15T09:00:00+07:00
- **Last Updated**: 2026-10-29T19:47:00+07:00

[STATUS: ARCHIVED]

> [!WARNING]
> **Archived Historical Notes**: Do not use this file as an active architectural specification. Refer to [AGENTS.md](../AGENTS.md) and [docs/INDEX.md](INDEX.md) for current production contracts.

---

Type dữ liệu được định nghĩa ở đâu ?

Từ requirement đến code đáng tin cậy
Vai trò và nguyên tắc khi code với AI → Từ yêu cầu đến đặc tả và prompt → Data, biến, type và biểu thức → Cấu trúc điều khiển: rẽ nhánh và vòng lặp → Hàm, kiểm thử và review code AI → Đánh giá việc dùng AI và củng cố kiến thức → Mở rộng: Số thực, hàm và mức độ tự động hóa của AI

1. Code với AI
Yêu cầu(Khi data sai thì xử lý ntn ….. Nếu ko nói rõ thì AI sẽ tự suy luận giải pháp) → Đặc tả (Input/Output/Test : Xử lý theo quy tắc nào, Xử lý lỗi ra làm sao, Xử lý trường hợp exception, Trả ra như thế nào, …) → Prompt (Ngữ cảnh + Ràng buộc, xác định + Ví dụ: giúp AI hiểu nhiệm vụ và gen code phù hợp) → AI đề xuất code → Kiểm chứng(Đọc: Đối chiếu cách xử lý với đặc tả + Dò vết: Theo dõi từng step thực hiện với input cụ thể nào đó + Chạy test(Baseline, So sánh kết quả, Bổ sung test trước khi deploy để tránh quá nhiều exception): Chưa đạt → Sửa và kiểm tra lại bước AI gen code) → Quyết định (Chấp nhận / Sửa/ Từ chối)

Luôn đặt câu hỏi Why/How

Unit Testing: Test từng file
….

2. Vai trò của dev khi có AI
Người đặc tả: Làm rõ yêu cầu và ràng buộc. Xác định đầu vào, output,test
Người đọc mã: Hiểu từng bước xử lý. Dò vết, phát hiện giả định ẩn (Code thực hiện thế nào)
Người quyết định: Chấp nhận, sửa hoặc từ chối. Chịu trách nghiệm với mã đã nhận (Có đủ cơ sở để sử dụng)
Người kiểm chứng: Viết và chạy test. Kiểm tra trường hợp biên, dữ liệu sai (Có đúng yêu cầu không ?)
Quy tắc làm việc với AI
1. Thử trước, hỏi sau
Tự phân tích yêu cầu và phác thảo cách giải
Dành vài phút suy nghĩ trước khi hỏi AI
2. Bậc thang trợ giúp
Làm rõ yêu cầu → Gợi ý hướng giải → Ví dụ tương tự → Code mẫu để review → Lời giải đầy đủ
3. giải thích được
Chỉ accept nếu ta hiểu
giải thích từng dòng và cách xử lý
4. Dữ liệu an toàn
Không đưa vào Prompt
Khóa API + Pass
Data cá nhân + Tài liệu bản quyền
5. Nhật ký AI (Log): Minh bạch với sự hỗ trợ AI
Ghi prompt, kết quả và test(đạt/chưa đạt)
Ghi quyết định: nhận, sửa hoặc từ chối

Năng lực cần đạt:
1. Khi tự thực hiện:
Đặc tả
Viết pseudocode
Hiểu biến, kiểu và biểu thức
Đọc, dò vết điều kiện và vòng lặp
Viết hàm, nhận diện tác dụng phụ
Chạy test và gỡ lỗi
Quyết định: chấp nhận, sửa, từ chối
2. Khi làm cùng AI
Yêu cầu gợi ý thêm test
Nhờ phản biện pseudocode
Phát hiện lỗi data và ép type


Phát hiện lỗi điều kiện và vòng lặp
Gen code theo đặc tả
Dùng test thất bại để yêu cầu sửa
Ghi nhật ký và lý do quyết định
Output cần đạt: Code + Đặc tả+ Test + pseudocode + Nhật ký AI + quyết định review



Bảng đặc tả có thể chạy:
Input: chuyenCan, baiTapLon, thi - kiểu number
Điều kiện: Mỗi điểm tổng [0; 10] , Điểm 0 hợp lệ
Xử lý: tổng = 10xchuyenCan + 50x……
	Điểm tổng kết = ….
Output: Điểm tổng kết: number ; Đầu ra sai: null
Xếp loại: ….
Thứ tự: Làm tròn trước → Xếp loại sau

Pseudocode: Hợp đồng giữa người và AI
Thiết kế (Độc lập với syntax) → Đưa vào Prompt (Ràng buộc các bước xử lý) → Đối chiếu(Kiểm tra code AI theo thuật toán)
Prompt có cấu trúc như thế nào ? 
- Viết file md, step by step (Nhưng không phải lúc nào AI cũng làm đúng theo)
- Khi prompt phải có rule bắt AI nếu không đủ data thì dừng lại việc giải quyết vấn đề đó chứ không phải bằng mọi giá giải quyết được

Cấu trúc Prompt cho AI chuẩn:
Ngữ cảnh: JS trên Node.js 22, dùng khai báo hàm; không thư viện ngoài
Nhiệm vụ: Viết tinhDiemTongKet(chuyenCan, baiTapLon, thi) và xepLoai(diem)
Ràng buộc: Điểm:number trong [0,10]; dữ liệu sai → null. Trọng số 10/50/40; làm tròn 1 chữ số thập phân. …..
Ví dụ: (9,8,7) → 7.7 ; (11,5,5) → null ; …..
Output: Trả source code và liệt kê các giả định đã sử dụng

Hỏi tiếp để kiểm chứng: Còn edge case nào chưa xử lý ? Đề xuất thêm 5 test ngoài danh sách ? ….

Quy trình kiểm chứng:
Chạy thử (typeof hoặc dữ liệu nhỏ) → Đọc tài liệu (MDN + Node.js + ECMAScript) → Kiểm tra version (Đúng môi trường đang dùng) → Xác minh nguồn (Mở và đọc liên kết AI đưa)