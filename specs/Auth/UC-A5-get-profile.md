# UC-A5 — Xem Hồ Sơ Người Dùng (Get Profile)

| | |
|---|---|
| **Use Case ID** | UC-A5 |
| **Tên** | Xem Hồ Sơ Người Dùng (Get Profile) |
| **Phiên bản** | 1.0 — 2026-07-10 |
| **Actor chính** | User đã đăng nhập |
| **Loại** | Read-only |
| **Phạm vi** | v1.0 |

---

## 1. Mô tả Sơ lược
Người dùng đã đăng nhập có thể xem thông tin cá nhân của mình: email, tên, avatar, phương thức xác thực (Google hoặc email/password), và ngày tạo tài khoản. Đây là chức năng **read-only**, không chỉnh sửa.

---

## 2. Actors & Precondition

| Yếu tố | Chi tiết |
|---|---|
| **Actor chính** | User đã đăng nhập thành công (có JWT hợp lệ) |
| **Precondition** | Tài khoản tồn tại và đã được xác minh (verified); user còn phiên đăng nhập hợp lệ |
| **Scope** | API đọc dữ liệu; giao diện hiển thị hồ sơ |

---

## 3. Main Flow

**Step 1:** User đã đăng nhập, điều hướng tới trang hồ sơ (thường là menu Settings / Profile).

**Step 2:** Client gửi request `GET /api/user/profile` kèm JWT trong header `Authorization: Bearer {token}`.

**Step 3:** Server xác thực JWT:
- JWT hợp lệ, chưa hết hạn → tìm User theo `userId` từ token
- User tồn tại → trả dữ liệu

**Step 4:** Server trả HTTP 200 với JSON payload:
```json
{
  "email": "user@example.com",
  "name": "Nguyễn Văn A",
  "avatar": "https://s3.../user-123-avatar.jpg",
  "googleId": null,
  "createdAt": "2026-06-15T10:30:00Z"
}
```

**Step 5:** Client nhận và render giao diện hiển thị hồ sơ.

---

## 4. Alternative Flows

### 4.1 — JWT không hợp lệ
- **Trigger:** User gọi `/api/user/profile` mà JWT bị thay đổi / không có / hết hạn
- **Response:** HTTP 401 Unauthorized, không trả dữ liệu (generic error message)
- **Kết quả:** Frontend điều hướng về trang đăng nhập

### 4.2 — User không tồn tại (mơ hồ)
- **Trigger:** JWT hợp lệ nhưng `userId` không tìm thấy trong DB (lỗi cơ sở dữ liệu hiếm)
- **Response:** HTTP 404 Not Found (hoặc HTTP 401, tùy chính sách bảo mật)
- **Kết quả:** Log lỗi; gợi ý user đăng nhập lại

### 4.3 — Tài khoản bị vô hiệu
- **Trigger:** User bị khóa / vô hiệu hóa sau khi đăng nhập
- **Response:** HTTP 403 Forbidden, thông báo "Tài khoản của bạn đã bị vô hiệu"
- **Kết quả:** Frontend hướng dẫn liên hệ hỗ trợ

---

## 5. Acceptance Criteria (Given–When–Then)

### AC-A5-1 — Lấy hồ sơ người đăng ký email/password
- **Given** user đã đăng ký email/password và xác minh email thành công
- **When** gọi `GET /api/user/profile` với JWT hợp lệ
- **Then** nhận HTTP 200 với:
  - `email`: đúng email đăng ký
  - `name`: tên đã nhập lúc đăng ký
  - `avatar`: URL avatar (nếu có, hoặc null/default)
  - `googleId`: **null** (vì không đăng ký Google)
  - `createdAt`: timestamp tài khoản tạo

### AC-A5-2 — Lấy hồ sơ người đăng nhập Google
- **Given** user đã đăng nhập Google thành công
- **When** gọi `GET /api/user/profile` với JWT hợp lệ
- **Then** nhận HTTP 200 với:
  - `email`: email từ Google
  - `name`: tên từ Google (hoặc rỗng nếu Google không cung cấp)
  - `avatar`: avatar từ Google (hoặc URL do Google cấp)
  - `googleId`: **không null** (giá trị sub từ Google)
  - `createdAt`: timestamp tài khoản tạo

### AC-A5-3 — Account linking (email + Google)
- **Given** user ban đầu đăng ký email, sau đó đăng nhập Google cùng email (account linking)
- **When** gọi `GET /api/user/profile` với JWT từ đăng nhập Google
- **Then** nhận HTTP 200 với:
  - `email`: đúng email liên kết
  - `googleId`: **không null** (vì đã link Google)
  - `name`, `avatar`: dùng data Google nếu khác rỗng, hoặc giữ từ lần đăng nhập trước
  - *Behavior chi tiết (priority data) được ghi ở UC-A3 Account Linking*

### AC-A5-4 — JWT hết hạn
- **Given** user có token JWT nhưng đã hết hạn (expired)
- **When** gọi `GET /api/user/profile` với JWT cũ
- **Then** nhận HTTP 401 Unauthorized, không lộ dữ liệu; frontend tự động điều hướng về đăng nhập

### AC-A5-5 — JWT không có trong request
- **Given** user gọi API nhưng không gửi Authorization header
- **When** gọi `GET /api/user/profile` không kèm JWT
- **Then** nhận HTTP 401 Unauthorized; không yêu cầu đồng đơn email (vì không biết user nào)

### AC-A5-6 — Giao diện hiển thị hồ sơ (UI)
- **Given** user đang xem trang Profile
- **When** page load (hoặc user nhấp refresh)
- **Then** giao diện:
  - Hiển thị Email (read-only, không chỉnh sửa ở v1)
  - Hiển thị Tên (read-only ở v1)
  - Hiển thị Avatar (ảnh hoặc default avatar)
  - Nếu auth bằng Google: "Đăng nhập qua Google" + Google icon
  - Nếu auth bằng email/password: "Đăng nhập qua email"
  - Hiển thị "Tạo vào ngày {createdAt}"
  - **Không có** nút "Chỉnh sửa", "Xóa tài khoản" ở v1

### AC-A5-7 — Isolation — user không thể xem hồ sơ user khác
- **Given** user A có JWT hợp lệ
- **When** user A cố gọi `GET /api/user/profile?userId=B` (tìm cách truyền userId khác)
- **Then** hệ thống bỏ qua parameter, chỉ trả hồ sơ user A (từ token); hoặc nhận 400 Bad Request nếu endpoint không chấp nhận query param
- **Outcome:** user A không bao giờ lộ dữ liệu user B

### AC-A5-8 — Dữ liệu không bao gồm sensitive
- **Given** user gọi `GET /api/user/profile`
- **When** server trả response
- **Then** payload **không chứa**: password hash, reset token, secret, internal ID nào không cần thiết
  - ✓ `email`, `name`, `avatar`, `googleId`, `createdAt` — OK
  - ✗ `passwordHash`, `verificationToken`, `stripeCustomerId` — không trả

---

## 6. Quy tắc & Ràng buộc

| Quy tắc | Mô tả |
|---|---|
| **Read-only ở v1** | Chức năng này chỉ đọc; không hỗ trợ cập nhật hồ sơ ở v1 (update sẽ là UC riêng biệt ở v1.1+) |
| **Cô lập theo user** | API chỉ trả hồ sơ của chính user gọi request, dựa trên JWT; không truy xuất được user khác |
| **JWT bắt buộc** | Không có flow ẩn danh; không có API public để lấy hồ sơ user khác |
| **Ngôn ngữ** | `name`, `avatar` có thể từ Google hoặc từ form đăng ký (không quy định format) |
| **Avatar null/default** | Nếu user không tải avatar riêng, có thể trả URL default avatar hoặc null; giao diện xử lý display |
| **Timestamp ISO 8601** | `createdAt` ở định dạng ISO 8601 UTC (ví dụ: `2026-06-15T10:30:00Z`) |

---

## 7. Data & Giả định

| Item | Chi tiết |
|---|---|
| **User fields được trả** | `email`, `name`, `avatar`, `googleId`, `createdAt` |
| **Giả định** | User đã xác minh email (verified=true) — nếu chưa xác minh, có thể cần flow riêng (không phạm vi UC-A5) |
| **Avatar storage** | Avatar được lưu trên S3 hoặc URL từ Google; API trả URL hoặc null |
| **googleId null nếu** | User không đăng ký/liên kết Google; giá trị là chuỗi Google sub nếu có |

---

## 8. Lưu ý Kỹ thuật

- **Endpoint:** `GET /api/user/profile` (Java Spring Boot)
- **Authentication:** Bearer token JWT (header `Authorization`)
- **Response format:** JSON
- **HTTP Status:** 200 (success), 401 (unauthorized), 404 (not found, hiếm), 500 (server error)
- **CORS:** Nếu frontend & backend khác origin → cấu hình CORS cho endpoint này
- **Caching:** Tuỳ; có thể cache profile ở client 5–10 phút (refresh khi đăng nhập)

---

## 9. Tham chiếu

| Tài liệu | Liên kết |
|---|---|
| Auth PRD | `PRD-docmate-v1.3.md` §3.1 (Auth) |
| UC-A1 | Đăng ký email/password |
| UC-A3 | OAuth Google |
| Product Backlog | `02-product-backlog.md` (E1 — Auth & Account) |
| Acceptance Criteria | `03-acceptance-criteria.md` (UC-A5) |
