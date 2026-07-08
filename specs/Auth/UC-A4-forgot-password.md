# UC-A4 — Quên mật khẩu

| Module | Story | AC | Test |
|---|---|---|---|
| Auth | US-1.4 (Should — sau v1) | `03` §US-1.4 | TC-1.4-01..03 |

**Actor:** người dùng quên mật khẩu · **Trigger:** chọn "Quên mật khẩu" · **Mức:** user goal

**Preconditions:** không (không cần đăng nhập).
**Postconditions — thành công:** mật khẩu mới có hiệu lực, link reset vô hiệu, đăng nhập được bằng mật khẩu mới. **— thất bại:** mật khẩu cũ giữ nguyên; phản hồi không lộ email có tồn tại hay không.

## Main flow
1. Người dùng nhập email tại màn "Quên mật khẩu".
2. Java LUÔN trả cùng thông điệp: "Nếu email tồn tại, hướng dẫn đã được gửi" — bất kể email có trong hệ thống hay không.
3. (Chỉ khi email tồn tại và có `password_hash`) Java sinh token reset **≤ 30 phút, dùng 1 lần** → gửi email chứa link.
4. Người dùng mở link còn hạn → form đặt mật khẩu mới (≥ 8 ký tự).
5. Java verify token → cập nhật hash mới → đánh dấu token đã dùng.
6. UI xác nhận, điều hướng đăng nhập; mật khẩu cũ hết hiệu lực ngay.

## Alternative / Exception flows
- **2a. Email không tồn tại:** dừng sau bước 2 — cùng thông điệp, không email gửi (chống dò email).
- **2b. Email tồn tại nhưng tài khoản GOOGLE thuần (`password_hash=null`):** cùng thông điệp trung tính ở UI; email gửi đi thay vào đó hướng dẫn "tài khoản này đăng nhập bằng Google" (không có link reset).
- **4a. Link quá 30 phút:** từ chối + nút yêu cầu link mới.
- **4b. Link đã dùng:** từ chối như 4a; token không tái sử dụng được dù mật khẩu đổi thất bại giữa chừng ở phía client.
- **3a. Yêu cầu reset dồn dập cùng email:** rate limit (mỗi email tối đa X yêu cầu / giờ); token mới vô hiệu token cũ chưa dùng.

## Business rules
- Không phân biệt được email tồn tại qua UI/timing (PRD §8 chống dò).
- Token reset dùng chung bảng token với UC-A1 (`type=RESET`).

## Data touched
`users` (update `password_hash`) · bảng `verification_tokens` (xem UC-A1).

## ⚠️ Open decisions
- X rate limit reset: **default 3 yêu cầu / email / giờ**.
- Đổi mật khẩu có vô hiệu JWT đang sống không: **default không** (stateless, UC-A2) — token cũ sống tới `exp`; ghi rõ đánh đổi.
