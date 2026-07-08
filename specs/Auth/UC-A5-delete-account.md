# UC-A5 — Xóa tài khoản và toàn bộ dữ liệu

| Module | Story | AC | Test |
|---|---|---|---|
| Auth | US-1.5 (Should — sau v1) | `03` §US-1.5 | TC-1.5-01 [MAND], 02 |

**Actor:** người dùng đã đăng nhập · **Trigger:** chọn "Xóa tài khoản" trong cài đặt · **Mức:** user goal — **hành vi hủy diệt, không hoàn tác**

**Preconditions:** đăng nhập hợp lệ.
**Postconditions — thành công:** toàn bộ ChatSession, Source, Chunk/embedding, Message, file S3 và User bị xóa vĩnh viễn khỏi cả 3 hệ (PostgreSQL, Qdrant, S3); tài khoản không đăng nhập được nữa. **— thất bại/hủy:** không gì bị xóa.

## Main flow
1. Người dùng mở "Xóa tài khoản" → hộp thoại nêu rõ hậu quả: số phiên, số nguồn sẽ mất vĩnh viễn.
2. Xác nhận bằng cách **gõ lại chính xác email** của mình.
3. Java kiểm khớp email → thực hiện xóa theo thứ tự an toàn (`07` §5.1): delete `users` trong PostgreSQL (cascade toàn chuỗi ChatSession → Source → Chunk/Message/IngestJob) → delete Qdrant theo payload `userId` → delete mọi object S3 của user (file gốc + Markdown).
4. Java trả xác nhận; client xóa JWT, điều hướng màn chào.
5. Mọi JWT còn sống của user từ đây gặp 401 (UC-A2 5b — user không còn).

## Alternative / Exception flows
- **3a. Gõ email sai:** từ chối, không xóa gì, cho nhập lại.
- **3b. Bước Qdrant hoặc S3 fail sau khi Postgres đã xóa:** chấp nhận theo nguyên tắc "thà thiếu còn hơn thừa" — vector mồ côi không truy xuất được (join Postgres loại), job đối soát dọn Qdrant; object S3 sót được job dọn orphan xử lý. Người dùng vẫn nhận xác nhận thành công (dữ liệu đã không thể truy cập).
- **2a. Người dùng đóng hộp thoại:** hủy, không tác dụng phụ.

## Business rules
- Cascade 3 nơi + thứ tự Postgres → Qdrant → S3 (`07` §5.1, ADR-001 §4).
- GDPR-ready: xóa là triệt để, mục tiêu tức thời, tối đa ≤ 24h cho phần job (PRD §8).

## Data touched
Toàn bộ: `users` + cascade (`08` §2) · Qdrant filter `userId` · S3 prefix của user.

## ⚠️ Open decisions
- Grace period (giữ N ngày trước khi xóa thật): **default KHÔNG ở v1** — xóa ngay, đúng thông điệp "không hoàn tác"; xét soft-delete nếu có yêu cầu người dùng thật.
- Yêu cầu nhập lại mật khẩu với tài khoản email/pass (bên cạnh gõ email): **default không** — AC chỉ yêu cầu gõ email; JWT hợp lệ là đủ xác thực.
