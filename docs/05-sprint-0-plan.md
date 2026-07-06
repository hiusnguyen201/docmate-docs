# DocMate — Sprint 0 Plan

| | |
|---|---|
| **Phiên bản** | 1.0 — 2026-07-06 |
| **Sprint** | Sprint 0 (2 tuần) |
| **Tham chiếu** | `02-product-backlog.md` v1.3 · `03-acceptance-criteria.md` v1.3 · `04-dor-dod.md` |
| **Capacity** | Solo dev · ~10 ngày làm việc · 16 SP (theo ước lượng velocity 15–18 SP, chưa hiệu chỉnh) |

---

## 1. Sprint Goal

> **"Toàn bộ stack DocMate khởi động bằng một lệnh `docker compose up`, và rủi ro lớn nhất — chất lượng MarkItDown với tiếng Việt — có kết luận Go / Go-with-limits / No-Go ghi thành ADR."**

Sprint 0 **không tạo tính năng người dùng thấy được**. Nó mua hai thứ: (1) nền tảng để mọi sprint sau chỉ tập trung vào nghiệp vụ; (2) thông tin để quyết định sớm — nếu MarkItDown No-Go với tiếng Việt thì đổi hướng khi chi phí đổi hướng còn rẻ nhất.

**Sprint thành công khi:** cả 4 story Done theo DoD, và ADR MarkItDown có kết luận rõ (kể cả kết luận xấu — spike trả lời được câu hỏi là spike thành công).

---

## 2. Sprint Backlog

### Tổng quan

| # | Story | SP | Ưu tiên trong sprint | Lý do thứ tự |
|---|---|---|---|---|
| 1 | US-0.1 — Môi trường dev Docker Compose | 5 | 1 | Mọi thứ khác chạy trên nền này |
| 2 | US-0.2 — Extraction Service `/extract` | 5 | 2 | Spike US-0.5 cần endpoint này để chạy file mẫu |
| 3 | US-0.5 — [SPIKE] MarkItDown tiếng Việt | 3 | 3 — **chạy ngay khi US-0.2 dựng xong khung** | Rủi ro lớn nhất, cần kết luận sớm nhất có thể |
| 4 | US-0.3 — Schema PostgreSQL + Qdrant collection | 3 | 4 | Không phụ thuộc spike, làm song song/cuối sprint |
| | **Tổng** | **16** | | |

> **Nguyên tắc xếp:** spike không đợi US-0.2 Done hoàn toàn — chỉ cần `/extract` nhận file và trả Markdown ở mức thô là bắt đầu chạy bộ file mẫu được. Phần còn lại của US-0.2 (mã lỗi phân loại, metadata đầy đủ) hoàn thiện song song.

### 2.1. US-0.1 — Môi trường dev Docker Compose (5 SP)

| Task | Ước lượng |
|---|---|
| T1. Skeleton 3 service: Golang (module + healthcheck endpoint), Spring Boot (init + actuator health), FastAPI (app + `/health`) | 0,5 ngày |
| T2. `docker-compose.yml`: 6 container (3 service + PostgreSQL + Qdrant + MinIO), network nội bộ, volume dữ liệu | 0,5 ngày |
| T3. Healthcheck cho cả 6 container (depends_on + condition) | 0,25 ngày |
| T4. Nginx reverse proxy — duy nhất container expose ra ngoài; route thử tới Java + Golang | 0,5 ngày |
| T5. Kiểm chứng kết nối chéo theo AC: Golang→Java, Java→Python, Java→Postgres/Qdrant, Golang→MinIO (script smoke test) | 0,25 ngày |
| T6. README hướng dẫn: yêu cầu máy, `.env.example`, lệnh chạy, lệnh kiểm tra | 0,25 ngày |

**Definition of Done cụ thể:** máy sạch chỉ cần Docker → `docker compose up` → 6/6 healthy; smoke test kết nối chéo pass; không port nào ngoài Nginx expose ra host.

### 2.2. US-0.2 — Extraction Service `POST /extract` (5 SP)

| Task | Ước lượng |
|---|---|
| T1. Endpoint `POST /extract` nhận `{s3Key}` hoặc `{url}`, tải file từ MinIO/S3 | 0,5 ngày |
| T2. Bọc MarkItDown: file → Markdown + metadata (định dạng, kích thước/số trang, thời gian xử lý) | 0,5 ngày |
| T3. Phân loại lỗi: `UNSUPPORTED_FORMAT` / `CORRUPTED_FILE` / lỗi tải file — trả 4xx có mã, không bao giờ 500 trần | 0,5 ngày |
| T4. Test: PDF hợp lệ → 200 + Markdown; file hỏng → 4xx đúng mã; định dạng lạ → 4xx đúng mã | 0,5 ngày |
| T5. Dockerize + nối vào compose, smoke test từ Java gọi sang | 0,25 ngày |

**Mốc mở khóa spike:** sau T1+T2 (ngày ~3–4 của sprint), US-0.5 bắt đầu chạy được.

> Lưu ý phạm vi: fetcher URL đầy đủ + chống SSRF thuộc US-2.4 (sprint sau). Ở US-0.2, nhánh `{url}` chỉ cần fetch cơ bản đủ dùng cho spike; **chưa** cam kết chống SSRF — ghi chú rõ trong code để không ai tưởng đã bảo vệ.

### 2.3. US-0.5 — [SPIKE] MarkItDown tiếng Việt (3 SP — timebox cứng)

**Câu hỏi spike phải trả lời:** *MarkItDown đọc tài liệu tiếng Việt đủ tốt để cam kết các định dạng trong PRD §3.3 không? Định dạng nào Go, định dạng nào phải giới hạn hoặc loại?*

| Task | Ước lượng |
|---|---|
| T1. Chuẩn bị bộ file mẫu tiếng Việt **thật** (không tự chế): PDF văn bản, PDF scan, DOCX, XLSX, PPTX, ảnh chụp văn bản (OCR), audio ghi âm giọng Việt — mỗi loại ≥ 2 file, độ khó khác nhau | 0,5 ngày |
| T2. Chạy toàn bộ qua `/extract`, chấm điểm theo rubric: đúng nội dung / đúng cấu trúc (heading, bảng) / lỗi dấu tiếng Việt | 1 ngày |
| T3. Viết báo cáo tỉ lệ đọc đúng theo loại + kết luận Go / Go-with-limits / No-Go từng định dạng → ghi thành **ADR (MarkItDown)** trong repo docs | 0,5 ngày |

**Kịch bản theo kết quả:**

| Kết luận | Hành động |
|---|---|
| Go | Giữ nguyên PRD §3.3, không việc gì thêm |
| Go-with-limits | Cập nhật PRD §3.3 + AC liên quan (giới hạn định dạng, ghi rõ trong UI) — commit `docs(prd): …` ngay trong sprint |
| No-Go (một phần) | Đưa quyết định "engine thay thế sau Extraction Service" thành item backlog mới, bàn ở Planning Sprint 1; các định dạng No-Go tạm loại khỏi phạm vi ingest |

> Timebox 3 SP là trần. Hết timebox mà chưa đủ dữ liệu → kết luận với dữ liệu đang có + ghi rõ độ tin cậy, không gia hạn.

### 2.4. US-0.3 — Schema PostgreSQL + Qdrant collection (3 SP)

| Task | Ước lượng |
|---|---|
| T1. Migration PostgreSQL (Flyway): User → ChatSession → Source → Chunk/Message, FK `ON DELETE CASCADE` đúng chuỗi | 0,5 ngày |
| T2. Qdrant collection + payload index: `userId`, `sessionChatId`, `sourceId` (+ `chunkId` trong payload) | 0,25 ngày |
| T3. Test theo AC: insert chunk + point → search filter `userId`+`sessionChatId` → đúng thứ tự similarity, chỉ đúng phiên/user | 0,5 ngày |
| T4. Test vector mồ côi: xóa chunk khỏi Postgres, point còn trên Qdrant → search + join lại Postgres theo `chunkId` → point bị loại | 0,25 ngày |
| T5. Test cascade: xóa Source → chunk/message biến mất (Postgres) + job xóa point theo payload `sourceId` chạy | 0,5 ngày |

**Ghi chú:** đây là chỗ hiện thực hóa nguyên tắc dual-write của PRD §5.1 ở mức schema — làm kỹ 3 test trên vì chúng là móng cho DoD "chạm dữ liệu người dùng" của mọi sprint sau. Nội dung quyết định Postgres+Qdrant sẽ chép thành **ADR-001 ở Nhóm 3** — sprint này chỉ cần code + test.

---

## 3. Capacity & lịch dự kiến

| | Tuần 1 | Tuần 2 |
|---|---|---|
| **Trọng tâm** | US-0.1 → khung US-0.2 → bắt đầu spike | Hoàn thiện US-0.2 → chốt spike + ADR → US-0.3 |
| **Mốc** | Cuối tuần 1: compose up 6/6 healthy, `/extract` chạy thô, spike đã chạy loạt file đầu | Giữa tuần 2: ADR MarkItDown chốt. Cuối tuần 2: 4/4 story Done, Sprint Review |

Tổng ước lượng task: ~8 ngày / 10 ngày capacity → dự phòng ~2 ngày cho phát sinh (dựng môi trường lần đầu luôn lòi việc vặt). Nếu tuần 1 trượt mốc, cắt trước: T4 US-0.1 (Nginx có thể lùi sang Sprint 1 vì dev local chưa cần) — **không** cắt spike.

## 4. Rủi ro & phụ thuộc của sprint

| Rủi ro / phụ thuộc | Ảnh hưởng | Ứng phó |
|---|---|---|
| Bộ file mẫu tiếng Việt chưa có sẵn | Chặn spike — item DoR | Chuẩn bị ở T1 của spike, bắt đầu gom file **ngay ngày đầu sprint** song song với US-0.1 |
| MarkItDown cần dependency hệ thống (ffmpeg cho audio, tesseract/OCR) làm image Python phình | Trễ US-0.2 | Chấp nhận image lớn ở dev; tối ưu image là việc của sprint CI/CD (US-0.4), không phải bây giờ |
| Máy dev yếu, 6 container + model OCR nặng | Trải nghiệm dev chậm | Giới hạn RAM per-container trong compose; Qdrant/MinIO đều nhẹ, nặng nhất là Extraction — chỉ bật khi cần |
| VPS | **Không phụ thuộc** — Sprint 0 chạy local hoàn toàn; VPS chỉ cần từ US-0.4 (Sprint 6 theo lộ trình) | Không làm gì |
| API key Claude/Voyage | **Chưa cần** ở Sprint 0 (pipeline contextual thuộc US-2.2) | Không làm gì |

## 5. Ngoài phạm vi Sprint 0 (chống scope creep)

Những thứ **cố tình không làm** dù "tiện tay":

- CI/CD (US-0.4) — Sprint 6; sprint này chỉ cần test chạy được bằng lệnh local.
- Upload Service logic thật (US-0.7) — Sprint 1; skeleton Golang ở US-0.1 chỉ cần healthcheck.
- Auth (E1) — Sprint 1–2.
- Chống SSRF cho nhánh URL của `/extract` — thuộc US-2.4.
- Bộ test vàng (US-0.6) — Sprint 4, cần pipeline chạy được trước.

## Changelog

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 2026-07-06 | Bản đầu — Sprint Goal + backlog 16 SP theo backlog v1.3 |
