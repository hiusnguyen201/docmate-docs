# UC-I3 — Nạp YouTube URL / ZIP / Audio

| Module | Story | AC | Test |
|---|---|---|---|
| Ingestion | US-2.5 (Should), US-2.6 (Could), US-2.7 (Could) — sau v1 | `03` §US-2.5/2.6/2.7 | TC-2.5-01..02 · TC-2.6-01 · TC-2.7-01 |

**Actor:** người dùng trong một phiên · **Trigger:** dán YouTube URL / upload ZIP / upload audio · **Mức:** user goal — ba biến thể trên cùng khung UC-I1/I2

> Cả ba dùng lại toàn bộ khung pipeline (chunk → contextual → embed → PG/Qdrant → READY) — spec này chỉ tả phần **khác biệt** so với UC-I1 (file) và UC-I2 (URL).

## Biến thể A — YouTube URL (US-2.5)
**Main flow (delta so với UC-I2):**
1. Java nhận URL, nhận diện là YouTube → Source `source_type=YOUTUBE`, `origin_url`.
2. Extraction lấy **transcript** video công khai (không tải video) → Markdown; metadata: tiêu đề, kênh, link.
3. Tiếp như UC-I1 bước 8–11.

**Exceptions:**
- **2a. Video không có transcript:** `FAILED` — "video không có phụ đề/transcript".
- **2b. Video private/bị chặn/age-restricted:** `FAILED` — "video không truy cập được" (phân biệt với 2a).

## Biến thể B — ZIP (US-2.6)
**Main flow (delta so với UC-I1):**
1. ZIP đi qua Golang → S3 như file thường (≤ 50MB cả gói).
2. Java (job) giải nén an toàn, duyệt từng entry: mỗi file **hợp lệ** (định dạng hỗ trợ) → tạo MỘT Source riêng của phiên, đi tiếp pipeline độc lập.
3. Sau xử lý: panel hiển thị các Source mới + **danh sách file bị bỏ qua** kèm lý do từng file.

**Exceptions:**
- **2a. Zip bomb / nén lồng sâu:** giới hạn tổng dung lượng giải nén + độ sâu; vượt → `FAILED` cả gói, lý do rõ.
- **2b. Entry có đường dẫn thoát thư mục (`../`):** bỏ qua entry, ghi vào danh sách bỏ qua, log bảo mật.
- **2c. ZIP có mật khẩu / hỏng:** `FAILED` cả gói, phân loại.
- **2d. ZIP rỗng hoặc không entry nào hợp lệ:** `FAILED` — "không có file hỗ trợ nào trong ZIP".

## Biến thể C — Audio (US-2.7)
**Main flow (delta so với UC-I1):**
1. File audio định dạng hỗ trợ qua Golang → S3.
2. Extraction transcribe (MarkItDown) → transcript thành nội dung Source.
3. Tiếp như UC-I1.

**Exceptions:**
- **2a. Định dạng audio không hỗ trợ / file hỏng:** `FAILED` phân loại như UC-I1 7a.
- **Giới hạn tiếng Việt theo ADR-002 (kết luận spike US-0.5):** nếu Go-with-limits → UI ghi rõ giới hạn NGAY tại điểm upload (AC US-2.7), không để người dùng phát hiện sau.

## Business rules
- ZIP: mỗi file con là Source độc lập — xóa/bật-tắt từng cái riêng (session-centric).
- Cả ba biến thể phụ thuộc kết luận ADR-002 về chất lượng MarkItDown theo loại.

## Data touched
Như UC-I1; ZIP: nhiều `sources` từ một upload; YouTube: `s3_key` null như URL.

## ⚠️ Open decisions
- Giới hạn giải nén ZIP: **default tổng ≤ 200MB sau giải nén, ≤ 50 file, không nén lồng (ZIP trong ZIP bị bỏ qua)**.
- Độ dài audio tối đa: **default 2 giờ** — hiệu chỉnh theo spike.
