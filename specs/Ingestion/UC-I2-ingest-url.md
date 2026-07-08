# UC-I2 — Nạp URL trang web (snapshot + chống SSRF)

| Module | Story | AC | Test |
|---|---|---|---|
| Ingestion | US-2.4 | `03` §US-2.4 | TC-2.4-01..05 [MAND: 02] |

**Actor:** người dùng trong một phiên · **Trigger:** dán URL vào panel phải · **Mức:** user goal

**Preconditions:** đăng nhập, phiên thuộc user.
**Postconditions — thành công:** nội dung trang tại thời điểm nạp (snapshot) thành Source `READY` như file, ghi `origin_url`. **— thất bại:** `FAILED` với lý do phân biệt được nguyên nhân; URL nguy hiểm bị từ chối kèm log bảo mật.

## Main flow
1. Người dùng dán URL http/https vào panel.
2. Frontend gửi Java (không qua Golang — không có file) → Java validate scheme http/https, tạo Source (`source_type=URL`, `origin_url`) `PROCESSING` + job.
3. Job gọi Extraction `/extract {url}` — **fetcher chạy cô lập tại Python** (PRD §4.2).
4. Fetcher kiểm SSRF: resolve DNS → IP phải public (chặn private/loopback/link-local/metadata); kiểm lại **sau mỗi redirect** (redirect cũng phải http/https + IP public); pin IP đã kiểm khi kết nối (chống DNS rebinding).
5. Fetch với giới hạn kích thước + timeout; kiểm content-type thuộc whitelist.
6. Nội dung → MarkItDown → Markdown + metadata (tiêu đề trang) → trả Java.
7. Từ đây như UC-I1 bước 8–11 (chunk → contextual → embed → PG/Qdrant → `READY`). Panel hiển thị Source dạng URL kèm link gốc.

## Alternative / Exception flows
- **2a. Scheme khác http/https** (`file://`, `ftp://`, `javascript:`…): từ chối ngay "URL không được hỗ trợ".
- **4a. IP private/loopback/metadata — trực tiếp, qua DNS, hoặc qua redirect:** từ chối `SSRF_BLOCKED`, thông điệp người dùng chung chung "URL không được hỗ trợ", **log bảo mật đầy đủ** (URL, IP resolve, chuỗi redirect).
- **4b. DNS rebinding (resolve lần kiểm ra public, lần fetch ra private):** vô hiệu nhờ pin IP ở bước 4; nếu phát hiện lệch → như 4a.
- **5a. Content-type ngoài whitelist:** dừng, `FAILED` lý do "loại nội dung không hỗ trợ" kèm content-type nhận được.
- **5b. Vượt giới hạn kích thước:** dừng ngay khi chạm ngưỡng (streaming check, không tải hết rồi mới kiểm), `FAILED` lý do rõ.
- **5c. Link chết (DNS fail, 404, timeout) vs cần đăng nhập (401/403/trang login):** `FAILED` với HAI lý do phân biệt được — "liên kết không truy cập được" vs "trang yêu cầu đăng nhập" (AC).
- **Redirect quá M lần:** dừng, `FAILED`.

## Business rules
- Snapshot tại thời điểm nạp — không tự refresh nội dung (PRD §4.2).
- SSRF là suite test bắt buộc chạy trong CI (`11` §5).

## Data touched
Như UC-I1 (không có file gốc trên S3 — chỉ Markdown snapshot; `s3_key` null, `markdown_s3_key` có).

## ⚠️ Open decisions
- Whitelist content-type: **default `text/html`, `text/plain`, `application/pdf`, `text/markdown`, `application/xhtml+xml`** — mở rộng theo nhu cầu thật.
- Giới hạn kích thước fetch: **default 20MB** (đồng bộ mục tiêu "nguồn văn bản ≤ 20MB").
- M redirect tối đa: **default 5**.
- JS-rendered page (SPA): **v1 không render JS** — fetch HTML tĩnh; trang rỗng nội dung → thường rơi vào 5c dạng "không trích xuất được nội dung"; ghi backlog headless browser nếu nhu cầu thật.
