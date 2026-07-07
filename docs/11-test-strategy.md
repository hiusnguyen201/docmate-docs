# DocMate — Test Strategy

| | |
|---|---|
| **Phiên bản** | 1.0 — 2026-07-06 |
| **Tham chiếu** | `04-dor-dod.md` · `12-test-cases.md` · `13-golden-test-set.md` · PRD §8 (NFR) |
| **Bối cảnh** | Solo dev — không QA riêng; test tự động + checklist là lưới an toàn duy nhất |

---

## 1. Nguyên tắc

1. **Test bám rủi ro, không bám coverage:** ưu tiên tuyệt đối cho 3 vùng chết người của DocMate — **cô lập dữ liệu**, **xóa triệt để**, **không bịa**. Coverage % không phải mục tiêu.
2. **AC là hợp đồng test:** mỗi scenario Given-When-Then trong `03-acceptance-criteria.md` phải map tới ≥ 1 test case trong `12-test-cases.md` (tự động hoặc tay có checklist).
3. **Chất lượng AI đo bằng số, không bằng cảm tính:** mọi thay đổi model/ngưỡng/chunking chạy qua bộ test vàng (`13-golden-test-set.md`) trước khi merge.
4. **Test chạy được từ sạch:** toàn bộ test suite chạy trên stack `docker compose up` từ máy trắng — điều kiện DoD.

## 2. Tầng test (test pyramid hiệu chỉnh)

| Tầng | Phạm vi | Công cụ | Khi chạy |
|---|---|---|---|
| **Unit** | Logic thuần từng service: chunking, validate file, map errorCode, tính token, parse citation | JUnit 5 (Java) · `go test` (Golang) · pytest (Python) | Mỗi lần build; local + CI |
| **Integration** | Service + hạ tầng thật (Testcontainers hoặc compose): repository ↔ Postgres, VectorStore ↔ Qdrant, upload ↔ MinIO, `/extract` ↔ MarkItDown | JUnit + Testcontainers · pytest + compose | Mỗi lần build; local + CI |
| **Contract (S2S)** | Hợp đồng API nội bộ 07 §2: Golang→Java callback (schema, 401/404/409), Java→Python `/extract` (schema, errorCode) | Test integration hai phía cùng một bộ fixture JSON chia sẻ trong repo | Khi hợp đồng đổi + CI |
| **E2E** | Luồng xuyên hệ thống trên full stack: upload → READY → hỏi → trích dẫn → xóa → hỏi lại | Script bash/HTTP (v1); Playwright cho UI xét sau khi UI ổn định | Trước khi đóng story liên quan; trước release |
| **AI eval** | Chất lượng RAG trên bộ test vàng | Eval harness (US-0.6) | Trước mọi thay đổi model/ngưỡng/chunking/contextual; trước release |

> Không có tầng test riêng cho hiệu năng ở v1 — chỉ đo first-token/p95 bằng metric khi chạy eval + E2E, so với mục tiêu NFR. Load test thật hoãn sau v1.

## 3. Bộ test bắt buộc (mandatory suite) — gắn với DoD

Ba nhóm dưới đây là điều kiện đóng story theo `04-dor-dod.md` §2.2, tập trung tại một suite riêng chạy được độc lập:

### 3.1. Cô lập dữ liệu (isolation)
- User A không đọc/hỏi được dữ liệu user B — qua **cả** API lẫn vector search.
- Phiên X không "nhìn" thấy nguồn phiên Y của cùng user.
- Mọi query mới thêm vào codebase: self-review rà filter `userId`/`sessionChatId` (checklist DoD).

### 3.2. Xóa triệt để (cascade 3 nơi)
- Xóa Source/Session/Account → kiểm **PostgreSQL** (cascade FK), **Qdrant** (delete theo payload), **S3** (object biến mất).
- Test đinh: **"xóa rồi hỏi lại"** — câu từng trả lời được, sau khi xóa nguồn phải ra "không tìm thấy".

### 3.3. Dual-write an toàn
- Vector mồ côi (Postgres xóa, Qdrant còn) bị loại khỏi kết quả search nhờ join bắt buộc.
- Chunk thiếu vector (ghi Postgres xong, Qdrant fail) không xuất hiện trong câu trả lời — và job đối soát re-embed được.
- Job đối soát: dọn đúng mồ côi, không dọn nhầm point hợp lệ.

## 4. Test dữ liệu & fixture

| Bộ | Nội dung | Dùng cho |
|---|---|---|
| **Fixture nhỏ** | Vài file text/PDF tí hon sinh trong repo test | Unit + integration, chạy nhanh |
| **Bộ file mẫu tiếng Việt** (từ spike US-0.5) | PDF văn bản/scan, DOCX, XLSX, PPTX, ảnh, audio — file thật | Integration `/extract` + hồi quy chất lượng extract |
| **Bộ test vàng** | Tập tài liệu cố định Việt + Anh + ≥ 40 câu hỏi 4 nhóm | AI eval — đặc tả tại `13-golden-test-set.md` |
| **Fixture contract** | Bộ JSON request/response chuẩn của API nội bộ | Contract test hai phía |

Quy ước: fixture nằm trong repo code (`/test-fixtures`), file lớn (audio, PDF scan) cân nhắc Git LFS; bộ test vàng có version riêng — đổi bộ = mọi kết quả cũ hết so sánh được, chỉ đổi có chủ đích.

## 5. Test theo loại rủi ro đặc thù

| Rủi ro | Cách test |
|---|---|
| **SSRF (US-2.4)** | Suite riêng: private IP, localhost, metadata endpoint, redirect về private, DNS rebinding (mô phỏng bằng resolver test); chạy trong CI như test thường |
| **Bịa/ảo giác** | Không test được bằng assert đơn lẻ → đo bằng bộ test vàng nhóm 2 (không có đáp án) với chỉ tiêu ≥ 95% từ chối đúng |
| **Mâu thuẫn giữa các đoạn** | Bộ test vàng nhóm 4 — đo "nêu đủ các bên", cùng nguồn và khác nguồn |
| **Lệch ngôn ngữ Việt↔Anh** | Case đo hai chiều trong bộ test vàng (13 §4) |
| **Idempotency callback** | Contract test: callback lặp cùng `s3Key` → 409, không tạo Source trùng |
| **Streaming từ chối nửa vời** | E2E: Gate 2 kích hoạt → client nhận thông điệp từ chối trọn vẹn |

## 6. Khi nào chạy gì (tóm tắt vận hành)

| Thời điểm | Chạy |
|---|---|
| Mỗi lần commit (local, trước push) | Unit + integration của phần sửa |
| Trước merge `main` (CI từ US-0.4; trước đó: lệnh local) | Toàn bộ unit + integration + contract + mandatory suite |
| Đóng story | Các test case map với AC của story (12) + mandatory nếu story chạm dữ liệu |
| Thay đổi model/ngưỡng/chunking/contextual | Bộ test vàng, so sánh với baseline gần nhất |
| Trước release v1 | Tất cả + E2E đầy đủ + bộ test vàng, lưu báo cáo vào repo docs |

## Changelog

| Phiên bản | Ngày | Thay đổi |
|---|---|---|
| 1.0 | 2026-07-06 | Bản đầu |
