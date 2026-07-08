# UC-P1 — Xem danh sách nguồn, preview Markdown, tổng quan phiên

| Module | Story | AC | Test |
|---|---|---|---|
| Sources | US-3.1 (phần xem), US-3.2, US-3.4 | `03` §US-3.1 sc.1, §US-3.2, §US-3.4 | TC-3.1-01 · TC-3.2-01 · TC-3.4-01 |

**Actor:** người dùng đang mở phiên · **Trigger:** mở panel phải / bấm vào một nguồn / nhìn khu tổng quan · **Mức:** user goal

**Preconditions:** đăng nhập, phiên thuộc user.
**Postconditions:** thấy đúng và đủ nguồn của **riêng phiên đó**; preview phản ánh Markdown đã trích xuất; số liệu tổng quan khớp dữ liệu thật.

## Main flow — danh sách
1. Mở phiên → panel phải tải danh sách Source của phiên: tên, loại (file/URL/YouTube), trạng thái, thời điểm thêm, cờ bật/tắt.
2. Danh sách cập nhật realtime theo UC-I4 (nguồn mới, đổi trạng thái).

## Main flow — preview
1. Bấm một nguồn `READY` → panel mở preview render Markdown đã trích xuất (tải từ `markdown_s3_key`).
2. Người dùng đối chiếu hệ thống "đọc" đúng chưa; đóng preview quay lại danh sách.

## Main flow — tổng quan
1. Khu tổng quan hiển thị: số nguồn, tổng dung lượng, tổng chunk đã index, ngôn ngữ phát hiện.
2. Số liệu tự cập nhật khi thêm/xóa nguồn hoặc nguồn đổi trạng thái (chunk chỉ tính nguồn `READY`).

## Alternative / Exception flows
- **Preview 1a. Nguồn `PROCESSING`/`FAILED`:** hiển thị trạng thái (+ lý do nếu FAILED, theo bảng UC-I4) thay vì nội dung.
- **Preview 1b. Markdown quá dài:** render lười/phân đoạn — không treo UI (chi tiết hiện thực).
- **Danh sách 1a. Phiên chưa có nguồn:** empty-state kèm hướng dẫn thêm file/dán URL (đồng bộ thông điệp UC-C2 flow 4).
- **Truy cập nguồn của phiên khác/user khác qua API:** 404 (cô lập, như UC-S1).

## Business rules
- Panel chỉ hiển thị nguồn của đúng phiên đang mở — không có thư viện chung (session-centric, PRD §3.1).

## Data touched
`sources` (select theo `session_chat_id`) · S3 (`markdown_s3_key` khi preview) · `chunks` (count cho tổng quan).

## ⚠️ Open decisions
- "Ngôn ngữ phát hiện": **default detect trên Markdown lúc ingest, lưu thành cột `language` trên `sources`** (bổ sung `08` khi hiện thực US-3.4); tổng quan hiển thị tập ngôn ngữ của các nguồn READY.
- Tổng quan tính realtime hay đọc từ cột đếm sẵn: **default query aggregate mỗi lần mở panel + cập nhật theo sự kiện** — tối ưu sau nếu chậm.
