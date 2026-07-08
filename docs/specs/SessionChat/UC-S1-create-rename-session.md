# UC-S1 — Tạo / đổi tên phiên chat, xem lại lịch sử

| Module | Story | AC | Test |
|---|---|---|---|
| SessionChat | US-4.4 (phần tạo/đổi tên/lịch sử) | `03` §US-4.4 scenario 1 | TC-4.4-01 |

**Actor:** người dùng đã đăng nhập · **Trigger:** bấm "Phiên mới" / đổi tên / chọn phiên cũ ở sidebar trái · **Mức:** user goal

**Preconditions:** đăng nhập hợp lệ.
**Postconditions:** phiên tạo/đổi tên phản ánh ngay ở sidebar; mở phiên cũ thấy đủ lịch sử tin nhắn + trích dẫn + token usage.

## Main flow — tạo phiên
1. Người dùng bấm "Phiên mới".
2. Java tạo ChatSession (`user_id` = người gọi, tên mặc định) → trả về, sidebar thêm phiên và focus vào nó.
3. Khung chat trống + panel phải trống hiển thị hướng dẫn thêm nguồn (nối UC-C2 flow 4 khi hỏi mà chưa có nguồn).

## Main flow — đổi tên
1. Người dùng sửa tên phiên tại sidebar (inline).
2. Java cập nhật `name` → phản ánh ngay, không reload.

## Main flow — mở lại phiên cũ
1. Người dùng chọn phiên trong sidebar (sắp theo `updated_at` giảm dần).
2. Java trả: toàn bộ Message theo thứ tự thời gian (nội dung, citations, token_usage, rating) + danh sách Source của phiên cho panel phải + tổng token phiên.
3. Citation trong tin nhắn cũ vẫn click được → panel nhảy tới đoạn gốc (UC-C1 bước 9) — miễn nguồn chưa bị xóa.

## Alternative / Exception flows
- **Đổi tên rỗng/toàn khoảng trắng:** từ chối, giữ tên cũ.
- **Mở phiên cũ 3a. Citation trỏ nguồn đã xóa (UC-P3/UC-S2):** citation hiển thị dạng vô hiệu ("nguồn đã xóa"), không click nhảy được — tin nhắn vẫn giữ nguyên văn.
- **Truy cập phiên không thuộc mình (đổi id trên URL/API):** 404 — không phân biệt "không tồn tại" với "không phải của bạn" (chống dò; cô lập `userId` tầng thấp nhất).

## Business rules
- Cô lập dữ liệu theo `userId` (PRD §8) — mọi query phiên đều filter.
- Lịch sử lưu bền vững (PRD §4.4).

## Data touched
`chat_sessions` (insert/update/select) · `messages`, `sources` (select khi mở lại).

## ⚠️ Open decisions
- Tên mặc định phiên mới: **default "Phiên mới — {dd/MM HH:mm}"**; tự đặt tên bằng LLM theo câu hỏi đầu: ngoài phạm vi v1, ghi backlog.
- Phân trang lịch sử phiên dài: **default tải 50 message gần nhất, cuộn lên tải thêm** — chốt lại khi hiện thực UI.
