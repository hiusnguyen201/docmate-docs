# UC-A1 — Đăng ký bằng email/mật khẩu

| Module | Story | AC | Test |
|---|---|---|---|
| Auth | US-1.1 | `03` §US-1.1 | TC-1.1-01..04 |

**Actor:** khách chưa có tài khoản · **Trigger:** chọn "Đăng ký" · **Mức:** user goal

**Preconditions:** không cần đăng nhập.
**Postconditions — thành công:** User tạo (`email_verified=false`, hash bcrypt/argon2), email xác minh đã gửi; sau khi xác minh: tài khoản kích hoạt, đăng nhập được. **— thất bại:** không bản ghi mới, không email gửi.

## Main flow
1. Khách nhập email + mật khẩu (≥ 8 ký tự) + xác nhận mật khẩu.
2. Frontend validate cơ bản (định dạng email, độ dài, khớp xác nhận) → gửi Core (Java).
3. Java kiểm email chưa tồn tại (so citext — không phân biệt hoa thường) → tạo User `email_verified=false`, mật khẩu băm bcrypt/argon2.
4. Java sinh token xác minh (một lần, có hạn) → gửi email chứa link xác minh.
5. UI báo "kiểm tra hộp thư để xác minh" — chưa cho vào app.
6. Khách click link còn hạn → Java verify token → `email_verified=true`, token vô hiệu.
7. UI xác nhận kích hoạt, điều hướng đăng nhập (→ UC-A2).

## Alternative / Exception flows
- **3a. Email đã tồn tại** (kể cả khác hoa thường): báo lỗi rõ "email đã được đăng ký", không tạo trùng, không lộ thêm thông tin. *(Chấp nhận lộ tồn tại ở luồng đăng ký — khác luồng đăng nhập/reset.)*
- **2a/3b. Mật khẩu < 8 ký tự:** chặn tại frontend VÀ tại Java (không tin client).
- **6a. Link hết hạn:** báo hết hạn + nút gửi lại email xác minh (token mới, token cũ vô hiệu).
- **6b. Link đã dùng:** báo "đã xác minh hoặc link không còn hiệu lực", điều hướng đăng nhập.
- **4a. Gửi email thất bại:** User vẫn tạo; UI cho "gửi lại email xác minh"; lỗi log phía server.
- **Chưa xác minh mà đăng nhập:** xem UC-A2 2b.

## Business rules
- Hash bcrypt/argon2, không bao giờ plaintext/MD5 (PRD §8).
- Email unique mức DB bằng citext (`08` bảng USERS).

## Data touched
`users` (insert; `auth_provider=EMAIL`) · bảng/token xác minh (⚠️ dưới).

## ⚠️ Open decisions
- Hạn link xác minh: **default 24h** (AC chỉ nói "còn hạn").
- Lưu token xác minh: **default bảng `verification_tokens` riêng** (id, user_id, type, expires_at, used_at) — dùng chung cho UC-A4; bổ sung `08` khi hiện thực US-1.1.
- Chưa xác minh có được đăng nhập hạn chế không: **default KHÔNG** — chặn hẳn tới khi xác minh.
