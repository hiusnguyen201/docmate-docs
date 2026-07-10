# UC-A2 — Đăng nhập Email/Mật khẩu

| Module | Story | AC | Test |
|---|---|---|---|
| Auth | US-1.2 | `03` §US-1.2 | TC-1.2-01..04 |

**Actor:** người dùng có tài khoản · **Trigger:** gửi form đăng nhập / bấm đăng xuất / JWT hết hạn giữa chừng · **Mức:** user goal

**Preconditions:** tài khoản tồn tại, `email_verified=true` (với luồng email/pass).
**Postconditions — thành công:** client giữ JWT có hạn; mọi API Java (chat/phiên/nguồn) lẫn Golang (upload) chấp nhận cùng token. **— thất bại:** không token nào phát hành; thông điệp lỗi không lộ email tồn tại hay không.

## Main flow
1. Người dùng nhập email + mật khẩu.
2. Java tìm user theo email, so hash mật khẩu.
3. Đúng → Java phát hành JWT (claims tối thiểu: `sub`=userId, `iat`, `exp`) ký bằng khóa dùng chung cơ chế với Golang (`07` §2.3).
4. Client lưu token, đính `Authorization: Bearer` vào mọi request tới Java và Golang.
5. Mỗi service tự verify token cùng cơ chế; hợp lệ → xử lý request.
6. Đăng xuất: client hủy token phía mình → điều hướng về đăng nhập.

## Alternative / Exception flows
- **2a. Sai mật khẩu HOẶC email không tồn tại:** cùng MỘT thông điệp chung chung ("email hoặc mật khẩu không đúng") — hai trường hợp không phân biệt được, kể cả qua thời gian phản hồi (so hash dummy khi email không tồn tại).
- **2b. Tài khoản chưa xác minh email:** báo riêng "cần xác minh email" + nút gửi lại (nối UC-A1 6a).
- **5a. JWT hết hạn / chữ ký sai / thuật toán `none`** — ở BẤT KỲ service nào: 401; frontend bắt 401 → xóa token → điều hướng đăng nhập.
- **5b. Token hợp lệ nhưng user đã bị xóa (UC-A5):** 401, xử lý như 5a.

## Business rules
- Một cơ chế ký/verify duy nhất cho Java + Golang, khóa qua env chung (`07` §2.3, `10` A2).
- JWT stateless — server không giữ session.

## Data touched
`users` (đọc) — không bảng phiên đăng nhập.

## ⚠️ Open decisions
- Thuật toán: **default HS256** (một secret chung, đơn giản cho 2 service cùng mạng nội bộ; nâng RS256 nếu về sau có service chỉ-verify không đáng tin giữ secret) — chốt khi hiện thực US-1.2, cập nhật `07` §2.3.
- TTL token: **default 24h**, không refresh token ở v1 (hết hạn → đăng nhập lại).
- Đăng xuất: **default không revoke phía server** (stateless thuần); hệ quả chấp nhận: token đã phát vẫn sống tới `exp` — ghi rõ, xét blacklist sau v1 nếu cần.
