# UC-A3 — Đăng nhập bằng Google OAuth (kèm account linking)

| Module | Story | AC | Test |
|---|---|---|---|
| Auth | US-1.3 | `03` §US-1.3 | TC-1.3-01..02 |

**Actor:** khách hoặc người dùng có tài khoản · **Trigger:** bấm "Đăng nhập với Google" · **Mức:** user goal

**Preconditions:** không.
**Postconditions — thành công:** đăng nhập bằng JWT như UC-A2; nếu email mới → User mới `email_verified=true`; nếu email trùng tài khoản email/pass → hai phương thức link về MỘT tài khoản (`auth_provider=BOTH`). **— thất bại:** không tài khoản mới, quay lại màn đăng nhập kèm lỗi.

## Main flow
1. Người dùng bấm "Đăng nhập với Google" → redirect sang Google (authorization code flow).
2. Google xác thực, redirect về callback của Java kèm code.
3. Java đổi code lấy token → lấy email (đã Google verify) + sub.
4. Email chưa có trong `users` → tạo User mới: `auth_provider=GOOGLE`, `email_verified=true`, `password_hash=null`.
5. Java phát hành JWT (như UC-A2 bước 3) → vào thẳng app, không bước xác minh.

## Alternative / Exception flows
- **4a. Email trùng tài khoản email/pass sẵn có:** KHÔNG tạo tài khoản thứ hai. Java link tự động: `auth_provider=BOTH`, giữ nguyên `password_hash` → phát JWT vào app + hiển thị thông báo một lần "tài khoản Google đã được liên kết với tài khoản sẵn có". Từ đây đăng nhập được bằng CẢ hai cách.
- **4b. Email trùng tài khoản đã là GOOGLE/BOTH:** đăng nhập bình thường (idempotent).
- **2a. Người dùng hủy trên màn Google / Google trả lỗi:** quay lại màn đăng nhập, thông báo trung tính, không log lỗi mức nghiêm trọng.
- **3a. Code không đổi được token (hết hạn/giả mạo):** như 2a + log bảo mật.
- **4c. Tài khoản email/pass trùng nhưng `email_verified=false`:** vẫn link và set `email_verified=true` (Google đã verify email đó — bằng chứng mạnh hơn).

## Business rules
- Một email = một tài khoản, mọi ngả (PRD §4.1: account linking là MỘT hành vi nhất quán).
- Không hỏi người dùng "có muốn link không" — link tự động + thông báo (AC).

## Data touched
`users` (insert hoặc update `auth_provider`/`email_verified`).

## ⚠️ Open decisions
- Có lưu Google `sub` để đối chiếu về sau: **default có**, cột `google_sub` nullable trên `users` — bổ sung `08` khi hiện thực.
- Tài khoản GOOGLE thuần muốn đặt mật khẩu (để thành BOTH chiều ngược): **ngoài phạm vi v1** — dùng UC-A4 sau khi có? Không: UC-A4 default từ chối user không có `password_hash` → ghi backlog nếu cần.
