# UC-S2 — Xóa phiên chat (= xóa toàn bộ nguồn của phiên)

| Module | Story | AC | Test |
|---|---|---|---|
| SessionChat | US-4.4 (phần xóa) | `03` §US-4.4 scenario 2 | TC-4.4-02 [MAND] |

**Actor:** người dùng đã đăng nhập · **Trigger:** bấm xóa phiên tại sidebar · **Mức:** user goal — **hành vi hủy diệt, không hoàn tác**

**Preconditions:** phiên thuộc user.
**Postconditions — thành công:** Source/Chunk/Message/IngestJob của phiên xóa khỏi PostgreSQL (cascade), vector xóa khỏi Qdrant (payload `sessionChatId`), file S3 của phiên xóa; phiên biến khỏi sidebar; nội dung phiên không bao giờ xuất hiện trong retrieval nữa. **— hủy/thất bại:** không gì bị xóa.

## Main flow
1. Người dùng bấm xóa phiên.
2. UI hiển thị hộp thoại cảnh báo nêu **đích danh con số**: *"Xóa phiên sẽ xóa vĩnh viễn N nguồn và toàn bộ hội thoại"* (N = số Source thực của phiên).
3. Người dùng xác nhận.
4. Java xóa theo thứ tự an toàn (`07` §5.1): delete ChatSession trong PostgreSQL (cascade Source → Chunk/IngestJob, Message) → delete Qdrant theo payload `sessionChatId` → delete các object S3 của phiên (file gốc + Markdown, theo danh sách `s3Key` thu thập TRƯỚC khi delete Postgres).
5. Sidebar gỡ phiên; nếu đang mở phiên đó → điều hướng về phiên gần nhất hoặc màn trống.

## Alternative / Exception flows
- **3a. Hủy ở hộp thoại:** không tác dụng phụ.
- **4a. Qdrant/S3 fail sau khi Postgres đã xóa:** như UC-A5 3b — an toàn theo thiết kế (mồ côi bị join loại; job đối soát + job dọn orphan xử lý phần còn sót). Người dùng vẫn nhận thành công.
- **4b. Phiên đang có nguồn `PROCESSING`:** vẫn cho xóa; IngestJob liên quan bị cascade — pipeline đang chạy khi ghi kết quả sẽ không tìm thấy Source → hủy nhẹ nhàng, không tạo dữ liệu mồ côi (pipeline luôn kiểm Source tồn tại trước bước ghi).
- **Xóa phiên không thuộc mình:** 404 như UC-S1.

## Business rules
- Phiên là **gốc sở hữu** của nguồn (session-centric, PRD §3.1) — xóa phiên = xóa nguồn, không có "chuyển nguồn sang phiên khác".
- Cascade 3 nơi ≤ 24h, mục tiêu tức thời (PRD §8) — test đinh "xóa rồi hỏi lại".

## Data touched
`chat_sessions` + cascade (`08` §2) · Qdrant filter `sessionChatId` · S3 theo danh sách `s3Key` của phiên.

## ⚠️ Open decisions
- Thu `s3Key` trước khi xóa Postgres: **default select danh sách vào bộ nhớ job trong cùng giao dịch xử lý** — chi tiết hiện thực, ghi để không quên thứ tự (xóa Postgres trước rồi mới hỏi danh sách là quá muộn).
