# UC-I1 — Upload file vào phiên (upload → pipeline contextual → READY)

| Module | Story | AC | Test |
|---|---|---|---|
| Ingestion | US-2.1, US-0.7, US-2.2 | `03` §US-2.1, §US-0.7, §US-2.2 | TC-2.1-01..03 · TC-0.7-01..06 · TC-2.2-01..07 |

**Actor:** người dùng trong một phiên chat · **Trigger:** kéo-thả/chọn file ở panel phải · **Mức:** user goal — luồng xương sống của Ingestion

**Preconditions:** đăng nhập hợp lệ, đang mở một phiên thuộc mình.
**Postconditions — thành công:** Source `READY`; chunk (text gốc + `contextualizedText`) trong PostgreSQL; vector trong Qdrant payload đủ `{userId, sessionChatId, sourceId, chunkId}`; nội dung hỏi được ngay câu kế tiếp; hội thoại đang gõ không gián đoạn. **— thất bại:** Source `FAILED` + lý do phân loại dễ hiểu, HOẶC bị chặn từ đầu không sinh Source; không rác vĩnh viễn trên S3.

## Main flow
1. Người dùng kéo-thả/chọn file tại panel phải — khung chat vẫn thao tác bình thường (async toàn tuyến).
2. Frontend validate sơ bộ (extension, size ≤ 50MB) → gửi multipart tới **Upload Service (Golang)** kèm JWT + `sessionChatId`.
3. Golang verify JWT; validate định dạng theo **extension + MIME thật** và size ≤ 50MB.
4. Golang **stream** file lên S3 (không buffer toàn bộ vào RAM) → nhận `s3Key`.
5. Golang callback Java `POST /internal/sources {s3Key, sessionChatId, userId, metadata}` với S2S token (`07` §2.1).
6. Java kiểm phiên thuộc user → tạo Source `PROCESSING` + IngestJob `QUEUED` → trả `201 {sourceId}` → Golang trả client → panel hiện Source `PROCESSING` (realtime — UC-I4).
7. Job chạy: Java tải file từ S3 → gọi Extraction `/extract {s3Key}` → nhận Markdown + metadata (lưu Markdown lên S3, ghi `markdown_s3_key`).
8. Java chunk theo heading (~500–1000 token, overlap 10–15%), ghi `position` + `location_hint`.
9. Với mỗi chunk: Claude sinh ngữ cảnh ~50–100 token (prompt caching phần toàn văn — exact-prefix, PRD §6.2) → ghép context + chunk → Voyage embed. Tiến trình % chunk đã xử lý đẩy về panel.
10. Ghi theo thứ tự dual-write (`07` §5.1): insert chunks vào PostgreSQL **trước** → upsert points vào Qdrant **sau**.
11. Source = `READY`; token contextual log vào IngestJob; panel cập nhật realtime. Câu hỏi kế tiếp trong phiên đã truy xuất được nguồn này.

## Alternative / Exception flows
- **2a/3a. File sai định dạng hoặc > 50MB:** chặn tại frontend (2a) VÀ tại Golang (3a — không tin client, kiểm MIME thật chống đổi đuôi); thông báo kèm danh sách định dạng hỗ trợ; **không gì lên S3, không Source nào sinh ra.**
- **3b. JWT sai/hết hạn:** 401, không gì lên S3 (nối UC-A2 5a).
- **5a. Callback fail (Java chết/timeout):** Golang retry N lần với backoff; hết N → dọn object S3 vừa đẩy (hoặc đánh dấu orphan cho job dọn) + client nhận lỗi rõ.
- **5b. Callback lặp (Golang timeout nhưng lần đầu đã thành công):** Java trả **409** theo `s3Key` — không Source trùng (idempotency, `07` §2.1); Golang coi 409 là thành công, trả client `sourceId` đã có.
- **6a. `sessionChatId` không tồn tại / không thuộc user:** Java trả 404 → Golang dọn S3 như 5a, client nhận lỗi.
- **7a. Extraction trả lỗi phân loại** (`UNSUPPORTED_FORMAT` / `CORRUPTED_FILE`...): không retry (lỗi vĩnh viễn) → Source `FAILED` + `failure_code`, panel hiện lý do dễ hiểu (UC-I4).
- **9a. Claude/Voyage lỗi tạm thời (timeout, 429):** retry backoff theo bước; quá N lần → `FAILED` + phân loại; **token đã tiêu vẫn log** vào IngestJob.
- **10a. Crash sau khi Postgres ghi xong, Qdrant chưa:** chunk thiếu vector = không truy xuất được (an toàn); job đối soát phát hiện → re-embed (TC-2.2-07).
- **11a. Nguồn lớn nhiều chunk vượt mục tiêu 2 phút:** không coi là lỗi — panel hiển thị tiến trình % thay vì treo (NFR).

## Business rules
- Validate TẠI Golang TRƯỚC khi lên S3 (PRD §4.2). Golang không chứa logic nghiệp vụ nào khác.
- Contextual embedding áp dụng từ v1, đánh đổi thời gian/chi phí ingest có chủ đích (PRD §6.2).
- Mọi chunk gắn đúng `sessionChatId` của phiên upload (cô lập — TC-2.2-05).

## Data touched
`sources`, `ingest_jobs`, `chunks` (`08`) · Qdrant upsert · S3: file gốc + Markdown.

## ⚠️ Open decisions
- N retry callback (Golang) và N retry pipeline (Java): **default 3 lần, backoff 1s/5s/25s** — config được.
- Giới hạn upload song song mỗi phiên: **default 5 file cùng lúc**, còn lại xếp hàng phía client.
- Job queue v1: **default queue in-process của Java (DB-backed qua bảng `ingest_jobs`)** — không thêm message broker; xét lại nếu cần scale (NFR khả năng mở rộng đã chừa cửa).
