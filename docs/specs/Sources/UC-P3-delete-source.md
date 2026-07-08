# UC-P3 — Xóa một nguồn khỏi phiên (cascade 3 nơi)

| Module | Story | AC | Test |
|---|---|---|---|
| Sources | US-3.1 (phần xóa) | `03` §US-3.1 sc.2–3 | TC-3.1-02 [MAND — "xóa rồi hỏi lại"], 03 |

**Actor:** người dùng đang mở phiên · **Trigger:** bấm xóa trên một nguồn · **Mức:** user goal — **hành vi hủy diệt, không hoàn tác**

**Preconditions:** nguồn thuộc phiên của user.
**Postconditions — thành công:** Source + chunks (PostgreSQL cascade) + vectors (Qdrant theo payload `sourceId`) + file S3 (gốc + Markdown) bị xóa (trong job, mục tiêu tức thời, tối đa ≤ 24h); nội dung nguồn không bao giờ xuất hiện trong retrieval nữa. **— hủy/thất bại:** không gì bị xóa.

## Main flow
1. Người dùng bấm xóa trên nguồn → hộp thoại xác nhận nêu tên nguồn + "xóa vĩnh viễn, không hoàn tác".
2. Xác nhận → Java thu `s3_key` + `markdown_s3_key` rồi xóa theo thứ tự (`07` §5.1): delete Source trong PostgreSQL (cascade `chunks`, `ingest_jobs`) → delete Qdrant theo payload `sourceId` → delete 2 object S3.
3. Panel gỡ nguồn; tổng quan (UC-P1) cập nhật số liệu.
4. Câu hỏi kế tiếp: nội dung nguồn đã xóa không truy xuất được — nếu không nguồn khác chứa đáp án → thông điệp từ chối chuẩn (UC-C2).

## Alternative / Exception flows
- **1a. Hủy ở hộp thoại:** không tác dụng phụ (TC-3.1-03).
- **2a. Qdrant/S3 fail sau khi Postgres xong:** an toàn theo thiết kế — join loại mồ côi ngay lập tức, job đối soát/dọn xử lý phần sót (như UC-S2 4a). Người dùng nhận thành công.
- **2b. Xóa nguồn đang `PROCESSING`:** cho phép; pipeline đang chạy tự hủy nhẹ nhàng (kiểm Source tồn tại trước mỗi bước ghi — như UC-S2 4b).
- **Message cũ có citation trỏ nguồn vừa xóa:** tin nhắn giữ nguyên văn; citation vô hiệu hóa hiển thị "nguồn đã xóa" (đồng bộ UC-S1).

## Business rules
- "Nguồn đã xóa không bao giờ xuất hiện trong retrieval sau đó" là **test bắt buộc** (PRD §4.3) — TC-3.1-02 là test đinh của cả hệ.
- Khác UC-P2: xóa là mất dữ liệu; bật/tắt là filter. UI phải phân biệt rõ hai hành động.

## Data touched
`sources` + cascade (`08` §2) · Qdrant filter `sourceId` · S3: 2 object.

## ⚠️ Open decisions
- Không mục mở — UC này phải cứng tuyệt đối; mọi chi tiết đã chốt ở `07` §5.1 và ADR-001.
